# Execution Units

## What a unit is

An execution unit is one resumable flow of control: a suspension state,
a wait record, and a reference the scheduler can queue. It is neither a
thread nor an actor. A thread runs whatever unit the scheduler mounts on
it; an actor owns memory and a mailbox and *runs on* a unit for the
duration of one message.

**A unit is an ordinary runtime object** — entity kind 0, with a class
the runtime builds, an `RcHeader` at offset zero, and a reference count
for its lifetime (`dev/DECISIONS.md`, 2026-08-12). An ordinary unit is
allocated in its thread's heap and never leaves that thread; an actor's
unit is allocated in the actor's arena and travels with the actor. The
stack and machine registers are a separate pooled object rather than part
of the unit.

**Only its own thread ever touches a unit.** A signal from another thread
is delivered to that thread's reactor, which runs there and touches the
unit on its behalf. This invariant is what the parking protocol below is
written against, and it is why that protocol needs no atomic operation.

The substrate owns three things about a unit and nothing else: how it
suspends, how it is resumed, and what it is waiting for. What it computes,
what memory it owns, and what it means to the consumer are the consumer's.

### One unit per message

An actor processes one message at a time, so at most one unit is live for
it. That unit's life is the message: it is created when the message is
taken from the mailbox and destroyed when the message is done. A request
is one message, so a request is one unit. The mailbox keeps filling while
the unit is parked, because an actor parked mid-message has a non-empty
stack and cannot begin another message.

### Actors are stackful

An actor always runs on a stackful unit. A synchronous call to another
actor parks the caller mid-message, and the frames below that point are
ordinary compiled frames — some of them foreign. Only a stackful unit can
suspend there.

Stackless units exist only where the substrate cooperates with the
compiler that produced them, which today means Limelight. A consumer
reaching the substrate over the C ABI gets stackful units and no choice
(`dev/DECISIONS.md`, 2026-08-12), because a state machine we did not
compile cannot be polled without an agreed layout, and a poll that meets
a foreign frame has nowhere to return `Pending` to.

## One handle, two kinds

The scheduler queues one thing: a counted reference to a unit. Whoever
arms a half of a wait holds one too, so the unit cannot be freed
underneath a wake that is still on its way. The suspension kind is a
field of the unit, which dispatch reads anyway.

The wake path never inspects the kind: it moves a reference to a run
queue. Dispatch reads the kind once, on mount, and branches to one of two
resume routines. Everything downstream of that branch differs; everything
upstream is common, which is what keeps the two kinds from becoming two
schedulers.

An earlier version of this document used a 64-bit word naming a pool
slot and a generation. It is retired: a coroutine is a refcounted object,
and the reference count does the job the generation was invented for
(`dev/DECISIONS.md`, 2026-08-12).

## Stackful units

A stackful unit owns a stack, taken from the stack allocator
(`design/stacks.md`). Resuming it switches the machine context to that
stack; suspending switches back to the worker. Between the two, the
unit's frames stay alive on its own stack, untouched by anything else.

Because the frames persist, a stackful unit may suspend at any depth,
including below a frame compiled by someone else. That is the property
foreign languages need, and it needs nothing from their compilers: a C
ABI and a context switch are enough. What the switch saves, and when it
may save less, is `design/switching.md`.

## Stackless units

A stackless unit is a state machine the compiler produced from a body
with suspension points in it, in the shape of a Rust `async fn`. Its
state lives in its own allocation, sized by that compiler; the worker
stack holds only the frames between two suspension points, and those
unwind every time the unit suspends.

### The ABI a compiler owes the substrate

The substrate never sees the state machine's fields. It needs this much
from the compiler that produced it:

| Item | Meaning |
|---|---|
| size, alignment | how much storage the unit's slot must provide |
| `poll(state, waker) -> Ready(value) \| Pending` | resume; run until the next suspension or completion |
| `drop(state)` | release the state and everything it owns |
| `is_thread_affine` | whether the state may hold thread-local addresses; see the pinning rule |

`poll` is an ordinary call: it returns to its caller, which is the worker.
Every other difference from a stackful resume follows from that.

### The waker the substrate hands to `poll`

`waker` is one function, callable from any thread:

```
wake(unit, half: u32, epoch: u64, result: Result)
```

The unit passes it to whatever will end the wait, together with a counted
reference to itself, which is what keeps it alive until the call arrives.
`half` names which entry of the wait record is being ended, `epoch` is
the value the record carried when that half was armed, and `result` is
what the unit reads after it resumes. The algorithm is in "Waking", and
it is the same call the reactor makes on a completion
(`design/reactor.md`).

### The worker stack is a bound, not a computation

A stackless unit runs on the worker's stack, so that stack must hold two
things at once: the deepest chain of ordinary calls between two
suspension points, and the depth of nested `poll` calls when futures are
composed, since an outer `poll` calls the inner one. Neither is derivable
from the source in general, because recursion and callbacks make it
unbounded. The worker stack is therefore guarded the way any thread stack
is, and overflow is a fault rather than a computed impossibility.

## Parking

Every suspension goes through one primitive, and its steps are ordered.
The order is still the protocol, but it no longer defends against another
thread: a unit is touched only by its own thread, so the steps are plain
stores and the wake that could once arrive mid-suspension cannot.

1. **Write the record.** A fresh `epoch`, the entries, the mode, and
   `remaining` for an AND wait.
2. **Arm each half.** Submit the operation, send the message, arm the
   timer — each carrying the unit, its half index, and the epoch from
   step 1. Arming after the record is written is what guarantees a
   completion finds somewhere to record itself.
3. **Suspend.** A stackful unit switches to the worker. A stackless unit
   returns `Pending` from `poll`.
4. **Mark `Parked`.** After the switch has completed, or after `poll` has
   returned, the worker stores `Parked`. It is the worker rather than the
   unit because a unit that announced itself parked while its machine
   context was still being saved would be resumable before it was
   savable, which is a bug even on one thread.

If a half completed inline during step 2 — the reactor's submission
returned a result at once, the mailbox already held the reply — the unit
does not suspend at all: step 3 is skipped and it continues.

Nothing else in the system may register a wait. A wait recorded outside
this primitive is invisible to the deadlock detector, and its absence is
not detectable from the outside (`dev/DECISIONS.md`, 2026-08-12).

**What this used to be, and why it changed.** Earlier versions gave every
transition a compare-and-swap, made the record a seqlock and had the
worker's swap fail when a wake had arrived. All of it existed to resolve
a race between a waker on another thread and a unit still suspending, and
that race does not exist: a completion returns on the ring of the worker
that submitted it, an actor's reply arrives as a message in its own
mailbox, and a signal from another thread goes to the reactor rather than
to the unit (`dev/DECISIONS.md`, 2026-08-12). The atomics come back the
day something else may wake a unit across a thread boundary, and the
retired protocol is the design for that day.

## Parking states

| State | Meaning |
|---|---|
| `Running` | mounted on a thread and executing |
| `Parked` | the suspension is finished and the unit is waiting |
| `Woken` | a wake has been accepted; the unit is owed one run queue slot |

Every transition is a plain store, because one thread performs all of
them. There were four states and a compare-and-swap on each while a wake
could arrive from another thread; that case is gone, and `Parking` went
with it.

| From | To | By |
|---|---|---|
| `Running` | `Parked` | the worker, after the suspension has finished |
| `Parked` | `Woken` | whoever ends the wait; it also enqueues |
| `Woken` | `Running` | the scheduler, mounting and resuming |

**Exactly one enqueue per wake**, because exactly one caller moves
`Parked → Woken` and a second finds `Woken` already set and returns.

## Waking

`wake(unit, half, epoch, result)` runs on the unit's own thread — the
reactor drains its worker's completions there, a mailbox is read by its
actor's thread, and a signal from elsewhere reaches the reactor rather
than the unit. It does five things in order.

1. **Validate the epoch.** Compare the record's epoch with the argument.
   A mismatch means this half was retired and the unit has since parked
   on something else, so the call returns. Without it a loser of a
   previous OR wait, firing between the win and the retirement, would
   wake the unit out of an unrelated wait with no half fired at all.

   The unit itself cannot be gone: whoever armed the half holds a
   counted reference to it, so it is alive for as long as the wake can
   arrive.
2. **Store the result** into the half's entry.
3. **Decide whether the wait is over.** Under **OR**, claim the winner
   field; a caller that finds it claimed returns. Under **AND**,
   decrement `remaining`; a caller that does not take it to zero returns.
4. **Retire the other halves** through their cancel handles. Retirement
   is asynchronous, so a retired half may still fire; the epoch check in
   step 1 is what makes that harmless.
5. **Move the state word** from `Parked` to `Woken` and enqueue.

Steps 1 through 4 are what make wakes safe to repeat. Step 5 is what
makes them arrive exactly once.

## The wait record

The wait record states what the unit is waiting for and who will end the
wait. It carries one entry per half:

| Field | Meaning |
|---|---|
| resource | the handle of what is being waited on: a mutex, a channel, an actor, an operation, a timer |
| cancel | an opaque handle that ends this half early |
| result | where the waker stores what the unit will read |
| fired | whether this half has already been satisfied |

**The record names a resource, not the unit that will end the wait.** Who
owes the wait is a field of the resource's own slot, read fresh when
anyone asks (`design/pool.md`). Naming a unit directly goes stale the
moment a resource changes hands: two units parked on one mutex both record
its holder, the holder releases it and finishes, its slot is reused, and
the second waiter's record now names a stranger.

and, once per record: the mode, `remaining` for AND, the winner field for
OR, and the `epoch` that every armed half carries.

**A resource says who owes it.** A mutex names its holder, an actor names
the unit processing its current message, an operation names the kernel, a
channel names its registered senders. `design/deadlock.md` reads those
fields; nothing here does.

**The mode:**

- **AND** — the unit continues when every half has fired. `remaining`
  starts at the number of halves.
- **OR** — the unit continues when any half has fired, and the winner
  retires the others.

`await` with a timeout is the ordinary OR: a timer and an operation.

Recording what a unit waits on, without anything saying who owes it, gives
half an edge, and half an edge cannot close a cycle. `php-src/ext/async`
records exactly that half (`dev/DECISIONS.md`, 2026-08-12). Here the other
half lives in the resource, which is what keeps it true as ownership
moves.

**Writers and readers.** The record is written and read by the unit's own
thread and by nothing else: the unit writes it while `Running`, and
whoever ends a wait writes one result and one counter, on that same
thread.

**The one reader from outside is the collector**, which is where deadlock
detection now lives (`dev/DECISIONS.md`, 2026-08-12). It reads the record
under the discipline the collector already has for reading a running
mutator's memory — block snapshots and a re-read of each recorded cell in
a later phase — rather than under a discipline of this document's
invention. The seqlock an earlier version put here was for a cross-thread
reader that does not exist.

## Mount and unmount

Mounting installs the consumer's context and unmounting removes it. The
substrate does not know what that context is: the consumer supplies a
mount hook and an unmount hook, and Limelight's hook installs the actor
context and puts the current arena into thread-local storage. A consumer
without actors installs whatever it has, or nothing.

The hooks fire when a unit is mounted on a thread and when it leaves,
never per `poll` of a stackless unit. A stackless unit installs no actor
context, because actors are stackful, so the hot path of polling carries
no hook at all.

### Unmount takes a reason

The unmount hook receives which event it is, because the two have
opposite meaning for the collector and a hook that cannot tell them apart
will answer a handshake at the wrong moment.

**`Boundary`** — the unit is finished: its stack is empty, the actor's
state is consistent, and the collector's handshake can be answered
(`rfc/runtime/actors.md`). This is the safepoint the Limelight collector
depends on.

**`Park`** — the unit is suspended mid-message with live frames, some of
them foreign. The actor's state is whatever the message left it, so no
handshake may be answered and no collection may run over its arenas.

Treating a park as a safepoint would let the collector scan an actor
halfway through a message. Treating a boundary as a park would leave the
scheduler holding a unit with nothing left to run.

### What a mid-message park costs the collector

An actor parked on an operation reaches no message boundary until that
operation completes, so it answers no handshake for as long as the
operation runs. Two consequences follow, and only the first is already
recorded elsewhere.

Deadlock detection cannot be built on the collector completing a cycle,
because a deadlocked actor never answers (`dev/DECISIONS.md`, 2026-08-12).

Mark termination waits on the slowest parked operation in the process. An
actor blocked on a socket for thirty seconds holds mark termination for
thirty seconds, and every actor's SATB buffer grows meanwhile. The
system-signal check that `rfc/runtime/actors.md` compiles into unbounded
loops does not help, because a parked unit executes no loop. This is a
defect of the actor model rather than of the substrate, and it belongs in
`rfc/BACKLOG.md`, which does not yet carry it — the substrate is only
where it becomes visible.

## Thread-local storage, and who may move

**Only an actor ever moves between threads.** An ordinary unit shares its
thread's memory and is allocated in that thread's heap, so it is pinned
for life — a consequence of where its memory is, not a scheduling policy
(`dev/DECISIONS.md`, 2026-08-12). Everything in this section is therefore
about actors, and about them only.

**Our rule:** no thread-local address is cached across a suspension
point. A unit reaches its context through the unit, not through a
register that happened to hold a TLS address before the switch. After a
move that address belongs to the previous thread, and the arena it names
belongs to another actor.

**The rule has no enforcer today.** rustc offers no way to state "do not
keep this TLS address across this call", and whether LLVM in fact hoists
a `thread_local!` address across a suspension point is not measured here.
What the design does instead is remove the opportunity: the context is
reached through the unit on every path that can suspend, and a
suspendable path carries no callee-saved registers at all
(`design/switching.md`), so nothing survives a park in a register to
begin with. That argument has not been checked against generated code,
which is why it is a research item rather than a settled one.

**Foreign frames are outside the rule entirely.** A C library suspended
below our frame holds thread-local state we neither see nor move:
`errno`, an OpenSSL error queue, a locale handle. Nothing of ours is
broken when that state is read on the wrong thread after a move; it was
simply left behind.

**When an actor may be re-mounted** is fixed in shape and open in detail
(`dev/DECISIONS.md`, 2026-08-12): only where it has stopped — reading its
mailbox, and waiting — and never while its foreign-frame counter is
non-zero. Which waits qualify, and what a consumer over the C ABI
declares about its own frames, is being researched. Until it closes, the
implementation re-mounts an actor only at a message boundary, which is
correct under every candidate answer.

## Migration and the single `unsafe impl Send`

Moving an actor between threads is sound because at most one thread
executes it at any moment, and the scheduler handoff that moves it
carries the release and acquire pair that publishes its memory
(`rfc/runtime/actors.md`). Rust cannot check this: the compiler cannot
see a stackful unit's frames, so `corosensei` marks its coroutine
`!Send` rather than guess.

The assertion lives in our code, in one place: the actor handle in the
scheduler carries a single `unsafe impl Send`, with the invariant above
written over it and the foreign-frame counter as its precondition. An
ordinary unit needs no assertion at all, because it never crosses.

A stackless unit written in Rust inside the substrate needs none either,
since rustc computes `Send` for its state machine the ordinary way. One
produced by the Limelight compiler is not covered by that: to rustc its
state is an opaque block behind size, alignment, `poll` and `drop`. Such
a unit crosses only when its producer declares that it may, and that
declaration is the second `unsafe` contract in the system.

## Restrictions on a stackless unit

A stackless unit may not host an actor, and may not suspend below a frame
it did not compile. Both come from the same fact: `poll` returns to its
caller, and there is no way to return `Pending` through a C frame that
expects a value.

Enforcement belongs to the compiler that produced the unit, and that is
why stackless units exist only where we cooperate with one. Limelight
rejects such a body statically, because it sees both the suspension point
and the call that crosses the ABI. The substrate performs no runtime check
for this, because it cannot see inside an opaque `poll`.

Anything that needs to suspend below a foreign frame is a stackful unit.
That is the whole reason the stackful kind is the default.

## Lifecycle

```
create ──▶ queued ──▶ Running ──┬──▶ Parking ──┬──▶ Parked ──▶ Woken ──┐
                        ▲       │              │                       │
                        │       │              └──▶ Woken ─────────────┤
                        │       │                                      │
                        │       └──▶ complete ──▶ teardown ──▶ released │
                        │                                              │
                        └──────────────────────────────────────────────┘
```

`Parking → Woken` is the wake that arrived during the suspension; the
worker enqueues on its failed swap. `Parked → Woken` is the ordinary
wake, enqueued by the waker. Both re-enter `Running` through the run
queue, and teardown is reached only from `Running`.

Creation takes a slot from the unit pool (`design/pool.md`) and, for a
stackful unit, a stack from the stack pool (`design/stacks.md`), then
enqueues the handle. That is a third enqueue, and it is not part of the
wake invariant: the unit has no wait record yet and no waker can name it.

Completion runs the consumer's unmount hook with reason `Boundary`,
releases the wait record, and returns both the slot and the stack. The
stack needs no further condition because nothing the kernel may touch
after submission lives on it: buffers and submission structures come from
the buffer pool (`design/stacks.md`, `design/reactor.md`). Without that
rule a late completion would write into a stack already handed to another
unit, and a stack would have to outlive its owner.

### Cancellation moves the state word, not a half

Cancelling a parked unit sets a cancelled bit in the state word with the
same compare-and-swap the parking protocol uses, and the winner bumps the
epoch so every armed half goes stale
(`design/cancellation.md`). It does not go through `wake`'s half index and
counter, because an AND wait's counter would swallow it: a waker that does
not take `remaining` to zero returns without waking, and a cancelled unit
would sleep forever waiting for halves that were just retired.

The state machine stays four states wide. The bit is a modifier on the
transitions that already exist, not a fifth state.

### A unit always ends itself

There is no forced teardown. Ending a unit means making its current wait
finish with a cancelled result and letting it unwind from its own
suspension point; no thread unwinds another thread's frames, because the
frames below a suspension point may be foreign and their cleanup is not
ours to run.

The consequence for any policy that picks a victim — a deadlock victim
above all — is that a unit parked below a foreign frame that never returns
cannot be ended at all. The cancel request reports that outcome rather
than failing silently, which is what lets such a policy pick another
victim instead of repeating itself (`design/cancellation.md`).

## Decided elsewhere

These documents are not yet written; the plan lists them in order
(`dev/PLAN.md`).

| Question | Document |
|---|---|
| what a context switch saves, and when it may save less | `design/switching.md` |
| stack reservation, commit, size classes, pooling | `design/stacks.md` |
| the unit slot layout, its alignment, and how pools are walked | `design/pool.md` |
| submitting operations, delivering completions, calling `wake` | `design/reactor.md` |
| retiring a half, and releasing what the kernel holds | `design/cancellation.md` |
| finding wait cycles, and the victim policy | `design/deadlock.md` |

## Open questions

- **Mark termination waits on the slowest parked operation.** Stated
  above; owned by the actor model and not yet recorded in
  `rfc/BACKLOG.md`.
- **Backpressure on a parked actor's mailbox.** A unit parked mid-message
  blocks its actor, so messages accumulate behind it. `rfc/BACKLOG.md`
  carries the question; the substrate is where the queue grows.
- **Pinning granularity.** A unit stays pinned while a foreign frame is
  live and becomes migratable when it returns. Whether that is enough
  depends on what the library left in thread-local storage after
  returning, which we cannot inspect. The conservative alternative — pin
  for life once a foreign frame has been entered — costs stealing for
  every unit that ever calls out, so the narrower rule is what this
  document states and the one that needs testing against real libraries
  first.
- **Enforcing the TLS rule.** Stated above: no mechanism, only the absence
  of a reason to violate it, plus a migration test that does not exist
  yet.
