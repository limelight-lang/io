# Execution Units

## What a unit is

An execution unit is one resumable flow of control: a suspension state,
a wait record, and a handle the scheduler can queue. It is neither a
thread nor an actor. A thread runs whatever unit the scheduler mounts on
it; an actor owns memory and a mailbox and *runs on* a unit for the
duration of one message.

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

The scheduler queues one handle type. A handle is a pointer to the unit
with the kind recorded in its low bits, which the unit slot's alignment
leaves free. One bit is used today; the slot is aligned to at least four
bytes so a second is available, and `design/pool.md` carries that as a
constraint on the slot layout rather than as an accident of it.

The wake path never inspects the kind: it moves a handle to a run queue.
Dispatch reads it once, on mount, and branches to one of two resume
routines. Everything downstream of that branch differs; everything
upstream is common, which is what keeps the two kinds from becoming two
schedulers.

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

`waker` is a pair of a data pointer and one function, callable from any
thread:

```
wake(data: *mut, half: u32, epoch: u64, result: Result)
```

The unit passes it to whatever will end the wait. `half` names which
entry of the wait record is being ended, `epoch` is the value the record
carried when that half was armed, and `result` is what the unit will
read after it resumes. The algorithm behind this call is in "Waking",
and it is the same call the reactor makes on a completion
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
The order is the protocol: a wait armed before the state moves would let a
completion arrive with nowhere to record itself.

1. **Enter `Parking`.** The unit stores `Parking` into its state word.
2. **Write the record.** Entries, mode, `remaining` for an AND wait, and a
   fresh `epoch`. Published with a release store, so a waker that reads
   the state word with acquire sees the record whole.
3. **Arm each half.** Submit the operation, send the message, arm the
   timer — each carrying the waker, its half index, and the epoch from
   step 2. Arming after step 1 is what makes `Running` unreachable for a
   waker.
4. **Suspend.** A stackful unit switches to the worker. A stackless unit
   returns `Pending` from `poll`.
5. **Publish `Parked` — from the worker, never from the unit.** After the
   switch has completed, or after `poll` has returned, the worker
   compare-and-swaps `Parking` to `Parked`. Publishing from the unit
   would announce a context that is still being saved, and a waker acting
   on that announcement would mount a half-saved unit on a second thread.

If the worker's compare-and-swap fails, the state is `Woken`: a wake
arrived during the suspension. The worker then enqueues the handle
itself. A stackful unit is already suspended and resumes normally. A
stackless unit is re-polled, which is correct because its frames unwound
when `poll` returned and its state machine holds everything it needs.

Nothing else in the system may register a wait. A wait recorded outside
this primitive is invisible to the deadlock detector, and its absence is
not detectable from the outside (`dev/DECISIONS.md`, 2026-08-12).

## Parking states

| State | Meaning |
|---|---|
| `Running` | mounted on a thread and executing |
| `Parking` | record written and halves armed; the suspension is not finished |
| `Parked` | the suspension is finished and no thread holds the unit |
| `Woken` | a wake has been accepted; the unit is owed one run queue slot |

Every transition is a compare-and-swap with acquire-release ordering. No
transition is a plain store, because two of them race by design:
`Parking → Parked` is attempted by the worker while `Parking → Woken` is
attempted by a waker.

| From | To | By | Then |
|---|---|---|---|
| `Running` | `Parking` | the unit | writes the record, arms the halves |
| `Parking` | `Parked` | the worker | nothing; the unit sleeps |
| `Parking` | `Woken` | a waker | nothing; the worker's failed swap enqueues |
| `Parked` | `Woken` | a waker | the waker enqueues |
| `Woken` | `Running` | the scheduler | mounts and resumes |

**Exactly one enqueue per wake.** The two enqueue paths are the worker's
failed swap and the waker's successful `Parked → Woken`, and the same
compare-and-swap decides which of them happens. A handle therefore cannot
be in two queues.

**A waker never observes `Running`.** The halves are armed in step 3, after
`Parking` is published in step 1, so nothing can call `wake` while the
state word still reads `Running`.

## Waking

`wake(data, half, epoch, result)` runs on whatever thread ended the wait,
and it does five things in order.

1. **Validate the epoch.** Read the record with acquire and compare its
   epoch with the argument. A mismatch means this half was retired and the
   unit has since parked on something else, so the call returns and does
   nothing. Without this check a loser of a previous OR wait, firing
   between the win and the retirement, would wake the unit out of an
   unrelated wait with no half fired at all.
2. **Store the result** into the half's slot in the record. This is the
   only write a waker makes to the record besides the counter below, and
   the state word's release on the next steps is what publishes it.
3. **Decide whether the wait is over.** Under **OR**, claim the win with a
   compare-and-swap on the winner field; a waker that loses the claim
   returns. Under **AND**, decrement `remaining`; a waker that does not
   take it to zero returns.
4. **Retire the other halves** through their cancel handles. Retirement is
   asynchronous, so a retired half may still fire; the epoch check in step
   1 is what makes that harmless.
5. **Move the state word.** `Parking → Woken` leaves the enqueue to the
   worker; `Parked → Woken` enqueues here. Finding `Woken` already set
   means another half arrived first under a mode that permits it, and the
   call returns.

Steps 1 through 4 are what make wakes safe to repeat. Step 5 is what makes
them arrive exactly once.

## The wait record

The wait record states what the unit is waiting for and who will end the
wait. It carries one entry per half:

| Field | Meaning |
|---|---|
| resource | an opaque identifier of what is being waited on |
| poster | who will end this half |
| cancel | an opaque handle that ends this half early |
| result | where the waker stores what the unit will read |

and, once per record: the mode, `remaining` for AND, the winner field for
OR, and the `epoch` that every armed half carries.

**The poster** is a unit, an actor, the kernel, or a set. A set is the
case of a channel with several registered senders, where no single party
owes the wake. `design/deadlock.md` treats a set edge as satisfiable by
any of its members, which is why the modes below matter to it.

**The mode:**

- **AND** — the unit continues when every half has fired. `remaining`
  starts at the number of halves.
- **OR** — the unit continues when any half has fired, and the winner
  retires the others.

`await` with a timeout is the ordinary OR: a timer and an operation.

Recording only the resource would give half an edge, and half an edge
cannot close a cycle. The `poster` field is what makes the record more
than a debugging aid, and it is the field `php-src/ext/async` does not
have (`dev/DECISIONS.md`, 2026-08-12).

**Writers and readers.** The unit writes the record while `Running`, in
step 2 of parking. A waker writes one result slot and one counter, in the
order above. The deadlock detector reads records of units it found in
`Parked` and writes nothing.

**The detector must validate, not merely observe.** Reading the state word
with acquire makes a record visible, and visibility is not stability: a
unit can wake, run, and park again on a different wait between two of the
detector's reads. A reader that spans more than one field re-reads the
epoch afterwards and discards the result if it changed. Without that, two
halves of two different waits combine into a cycle that never existed.

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

## Thread-local storage and pinning

**Our rule:** no thread-local address is cached across a suspension point.
A unit reaches its context through the unit, not through a register that
happened to hold a TLS address before the switch. After migration that
address belongs to the previous thread, and the arena it names belongs to
another actor.

**The rule has no enforcer today.** The substrate's own Rust is compiled
by rustc, and rustc offers no way to state "do not keep this TLS address
across this call". Whether LLVM in fact hoists a `thread_local!` address
across a suspension point is not measured here, and that is the point: a
rule that can be neither enforced nor checked is the contract `may` states
openly by marking spawn unsafe. What the design does instead is remove the
opportunity — the context is reached through the unit on every path that
can suspend, so no correct path needs a TLS address to survive a switch.
A test that migrates a unit across threads between every pair of
suspension points is owed at implementation.

**Foreign frames are outside the rule entirely.** A C library suspended
below our frame holds thread-local state we neither see nor move:
`errno`, an OpenSSL error queue, a locale handle. Nothing of ours is
broken when that state is read on the wrong thread after migration; it was
simply left behind.

**Pinning.** A unit is pinned to its thread while it is thread-affine, and
the scheduler does not steal a pinned unit. Two things make a unit
thread-affine:

- a live foreign frame, counted by the same depth counter that decides
  how much a context switch must preserve (`design/switching.md`),
  incremented on entry to foreign code and decremented on return. It is a
  counter rather than a flag because foreign code calls back into ours,
  and the inner call's return would otherwise clear a marker that the
  outer frame still needs;
- a consumer that declares its own frames thread-affine, per unit at
  creation. A consumer whose compiler we do not own knows what its code
  keeps in thread-local storage, and the substrate does not. The default
  for a C-ABI consumer is affine, because assuming otherwise breaks
  `errno` silently; Limelight declares its own units migratable.

The default costs work stealing for a consumer that never revisits it,
which is the price of being wrong in the safe direction. A consumer that
wants stealing declares it and takes the obligation.

## Migration and the single `unsafe impl Send`

Migration is sound here because at most one thread executes a unit at any
moment, and the scheduler handoff that moves it carries the release and
acquire pair that publishes its memory (`rfc/runtime/actors.md`). Rust
cannot check this: the compiler cannot see a stackful unit's frames, so
`corosensei` marks its coroutine `!Send` rather than guess.

The assertion lives in our code, in one place: the handle type in the
scheduler carries a single `unsafe impl Send`, with the invariant above
written over it and the pinning rule as its precondition.

A stackless unit written in Rust inside the substrate needs no assertion,
because rustc computes `Send` for its state machine the ordinary way. A
stackless unit produced by the Limelight compiler is not covered by that:
to rustc its state is an opaque block behind size, alignment, `poll` and
`drop`, and no inference applies to it. Such a unit is migratable only
when its producer declares it so through `is_thread_affine`, and that
declaration is the second `unsafe` contract in the system. There are
exactly two, and both are stated at the point where a unit crosses into
the scheduler.

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

### Cancellation leaves `Parked` the way a wake does

Cancelling a parked unit is a wake carrying a cancelled result, delivered
through the same `wake` call with the same epoch check. No state and no
path is added for it: the unit resumes, observes the cancelled result, and
unwinds from its own suspension point. This is what keeps the state
machine four states wide.

### Cooperative completion and forced teardown

A unit normally ends by finishing its body. Forced teardown — ending a
unit that has not agreed to end — is available only where the frames can
be unwound: unwinding through a foreign frame is undefined unless that
frame was compiled to allow it, and the cleanup it would skip is
arbitrary.

A unit with a live foreign frame is therefore not force-killable, only
cancellable at its next suspension point. A unit parked below a foreign
frame that never returns cannot be ended at all. Any policy that picks a
victim — a deadlock victim above all — inherits this limit and must state
what it does when the victim it wants is unkillable.

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
