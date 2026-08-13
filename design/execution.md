# Coroutines

## What a coroutine is

A coroutine is one resumable flow of control: a suspension state,
a wait record, and a reference the scheduler can queue. It is neither a
thread nor an actor. A thread runs whatever coroutine the scheduler mounts on
it; an actor owns memory and a mailbox and *runs on* a coroutine for the
duration of one message.

**A coroutine is an ordinary runtime object** — entity kind 0, with a class
the runtime builds, an `RcHeader` at offset zero, and a reference count
for its lifetime (`dev/DECISIONS.md`, 2026-08-12). An ordinary coroutine is
allocated in its thread's heap and never leaves that thread; an actor's
coroutine is allocated in the actor's arena and travels with the actor. The
stack and machine registers are a separate pooled object rather than part
of the coroutine.

**Only its own thread ever touches a coroutine.** A signal from another thread
is delivered to that thread's intake queue, drained there, and touches the
coroutine on its behalf. This invariant is what the parking protocol below is
written against, and it is why that protocol needs no atomic operation.

The substrate owns three things about a coroutine and nothing else: how it
suspends, how it is resumed, and what it is waiting for. What it computes,
what memory it owns, and what it means to the consumer are the consumer's.

### One coroutine per message

An actor processes one message at a time, so at most one coroutine is live for
it. That coroutine's life is the message: it is created when the message is
taken from the mailbox and destroyed when the message is done. A request
is one message, so a request is one coroutine. The mailbox keeps filling while
the coroutine is parked, because an actor parked mid-message has a non-empty
stack and cannot begin another message.

### Actors are stackful

An actor always runs on a stackful coroutine. A synchronous call to another
actor parks the caller mid-message, and the frames below that point are
ordinary compiled frames — some of them foreign. Only a stackful coroutine can
suspend there.

Stackless coroutines exist only where the substrate cooperates with the
compiler that produced them, which today means Limelight. A consumer
reaching the substrate over the C ABI gets stackful coroutines and no choice
(`dev/DECISIONS.md`, 2026-08-12), because a state machine we did not
compile cannot be polled without an agreed layout, and a poll that meets
a foreign frame has nowhere to return `Pending` to.

## One reference, two kinds

The scheduler queues one thing: a counted reference to a coroutine. Whoever
arms an entry of a wait holds one too, so the coroutine cannot be freed
underneath a wake that is still on its way. That reference is in exactly
one place at a time: the resource's waiter cell while the entry is armed —
an operation, timer or socket slot (`design/pool.md`), a waiter node of a
channel (`design/channels.md`) — and the wake payload afterwards, dropped
by whoever consumes it. During a pending cross-thread retire there are
transiently two: the cell's and the request's own, and both come home to be
dropped on the coroutine's thread (`design/cancellation.md`). The suspension
kind is a field of the coroutine, which dispatch reads anyway.

The wake path never inspects the kind: it moves a reference to a run
queue. Dispatch reads the kind once, on mount, and branches to one of two
resume routines. Everything downstream of that branch differs; everything
upstream is common, which is what keeps the two kinds from becoming two
schedulers.

An earlier version of this document used a 64-bit word naming a pool
slot and a generation. It is retired: a coroutine is a refcounted object,
and the reference count does the job the generation was invented for
(`dev/DECISIONS.md`, 2026-08-12).

## Stackful coroutines

A stackful coroutine owns a stack, taken from the stack allocator
(`design/stacks.md`). Resuming it switches the machine context to that
stack; suspending switches back to the worker. Between the two, the
coroutine's frames stay alive on its own stack, untouched by anything else.

Because the frames persist, a stackful coroutine may suspend at any depth,
including below a frame compiled by someone else. That is the property
foreign languages need, and it needs nothing from their compilers: a C
ABI and a context switch are enough. What the switch saves, and when it
may save less, is `design/switching.md`.

## Stackless coroutines

A stackless coroutine is a state machine the compiler produced from a body
with suspension points in it, in the shape of a Rust `async fn`. Its
state lives in its own allocation, sized by that compiler; the worker
stack holds only the frames between two suspension points, and those
unwind every time the coroutine suspends.

### The ABI a compiler owes the substrate

The substrate never sees the state machine's fields. It needs this much
from the compiler that produced it:

| Item | Meaning |
|---|---|
| size, alignment | how much storage the state's own allocation must provide |
| `poll(state, waker) -> Ready(value) \| Pending` | resume; run until the next suspension or completion |
| `drop(state)` | release the state and everything it owns |
| `is_thread_affine` | whether the state may hold thread-local addresses; see the pinning rule |

`poll` is an ordinary call: it returns to its caller, which is the worker.
Every other difference from a stackful resume follows from that.

### The waker the substrate hands to `poll`

`waker` is one function, callable from any thread:

```
wake(coroutine, entry: u32, epoch: u64, result: Result)
```

The coroutine passes it to whatever will end the wait, together with a counted
reference to itself, which is what keeps it alive until the call arrives.
`entry` is the index of the wait-record entry being ended, `epoch` is
the value the record carried when that entry was armed, and `result` is
what the coroutine reads after it resumes. The algorithm is in "Waking", and
it is the same call the reactor makes on a completion
(`design/reactor.md`).

### The worker stack is a bound, not a computation

A stackless coroutine runs on the worker's stack, so that stack must hold two
things at once: the deepest chain of ordinary calls between two
suspension points, and the depth of nested `poll` calls when futures are
composed, since an outer `poll` calls the inner one. Neither is derivable
from the source in general, because recursion and callbacks make it
unbounded. The worker stack is therefore guarded the way any thread stack
is, and overflow is a fault rather than a computed impossibility.

## Parking

Every suspension goes through one primitive, and its steps are ordered.
The order is still the protocol, but it no longer defends against another
thread: a coroutine is touched only by its own thread, so the steps are plain
stores and the wake that could once arrive mid-suspension cannot.

1. **Write the record.** A fresh `epoch`, the entries, the mode, and
   `remaining` for an AND wait. This is the only place the epoch is
   written: no path that ends a wait touches it (`dev/DECISIONS.md`,
   2026-08-13).
2. **Arm each entry.** Submit the operation, send the message, arm the
   timer — each carrying the coroutine, its entry index, and the epoch from
   step 1. Arming after the record is written is what guarantees a
   completion finds somewhere to record itself.
3. **Suspend.** A stackful coroutine switches to the worker. A stackless coroutine
   returns `Pending` from `poll`.
4. **Mark `Parked`.** After the switch has completed, or after `poll` has
   returned, the worker stores `Parked` — unless the cancelled bit is set, in
   which case it stores the cancellation in the record's own result slot,
   claims the winner with the reserved no-entry value, retires the entries,
   stores `Woken` and enqueues instead (`design/cancellation.md`). It claims
   before it retires for the same reason the cancel path does: retirement is
   asynchronous, and an entry armed at step 2 that fires afterwards is
   rejected by the claim and by nothing else, the epoch being the one just
   written. Retiring there is what unlinks the nodes this
   wait put into resource queues; a coroutine that leaves them behind is a channel
   waking a coroutine that has since parked on something else. It is the worker rather than the
   coroutine because a coroutine that announced itself parked while its machine
   context was still being saved would be resumable before it was
   savable, which is a bug even on one thread.

If an entry completed inline during step 2 — the reactor's submission
returned a result at once, the mailbox already held the reply — the coroutine
does not suspend at all: step 3 is skipped and it continues. **When an inline
satisfaction ends the wait, it retires every entry armed before it, its own
included** — under OR mode, and under AND when it was the last outstanding
entry. Leaving one armed would leave a resource holding a waiter for a wait that
no longer exists, and the wake it later sends is dropped on the epoch without
anyone else being woken (`design/channels.md`). **Under AND with entries still
outstanding nothing is retired and the coroutine suspends as usual**: retiring there
would leave the wait alive with nothing able to end it.

Nothing else in the system may register a wait. A wait recorded outside
this primitive is invisible to the deadlock detector, and its absence is
not detectable from the outside (`dev/DECISIONS.md`, 2026-08-12).

**What this used to be, and why it changed.** Earlier versions gave every
transition a compare-and-swap, made the record a seqlock and had the
worker's swap fail when a wake had arrived. All of it existed to resolve
a race between a waker on another thread and a coroutine still suspending, and
that race does not exist: a completion returns on the ring of the worker
that submitted it, an actor's reply arrives as a message in its own
mailbox, and a signal from another thread goes to the reactor rather than
to the coroutine (`dev/DECISIONS.md`, 2026-08-12). The atomics come back the
day something else may wake a coroutine across a thread boundary, and the
retired protocol is the design for that day.

## Parking states

| State | Meaning |
|---|---|
| `Running` | mounted on a thread and executing |
| `Parked` | the suspension is finished and the coroutine is waiting |
| `Woken` | a wake has been accepted; the coroutine is owed one run queue slot |
| `Terminal` | teardown has begun; nothing will run on this coroutine again |

Every transition is a plain store, because one thread performs all of
them. There were four states and a compare-and-swap on each while a wake
could arrive from another thread; that case is gone, and `Parking` went
with it.

**`Terminal` is not a parking state**: the first three describe a coroutine that
still has code to run, and this one says it has none. It exists because the
object outlives its own completion — whoever armed an entry holds a counted
reference, so a finished coroutine is still there to be asked — and because
`design/deadlock.md` distinguishes a terminated holder from a running one
in three of its liveness rows. No generation answers that question now
(`dev/DECISIONS.md`, 2026-08-12).

**It is stored at one point of teardown**: after the coroutine's own code has
finished — the stack unwound, or `drop(state)` returned for a stackless
coroutine — and before the wait record is released. Earlier would report a coroutine
as terminated while a guard in its frames still holds a mutex, and
`design/deadlock.md` treats a terminated holder as one that never releases,
so an ordinary unwind would be read as a deadlock. Later would let a
retired completion pass step 0 and read the epoch out of a record that has
been freed.

**`Woken` means the wait is decided**, and that equivalence is load-bearing
elsewhere: whoever moves `Parked → Woken` has already claimed the winner of
an OR wait or taken `remaining` to zero, so a later signal for the same wait
finds nothing to do. The delivery rule of `dev/DECISIONS.md`, 2026-08-13,
drops a signal on this ground, and step 12 of `design/deadlock.md` requires
the state to still be `Parked` for the same reason.

**An actor that declares mid-message movement carries a wider word**, and
only such an actor does (`dev/DECISIONS.md`, 2026-08-13). It holds the state
together with the owning worker — `Running(W)`, `Parked(W)`, `WokenLocal(W)`,
`WokenShared`, `Terminal` — its transitions are atomic stores rather than
plain ones, and the cancelled bit moves to a byte beside it. Everything below
reads as written for every other coroutine; for a declared one, "the coroutine's own
thread" means the thread the word names.

| From | To | By |
|---|---|---|
| `Running` | `Parked` | the worker, after the suspension has finished |
| `Parked` | `Woken` | whoever ends the wait; it also enqueues |
| `Woken` | `Running` | the scheduler, mounting and resuming |
| `Running` | `Terminal` | the worker, once the coroutine's own code has finished; nothing leaves `Terminal` |

**Exactly one enqueue per wake**, because exactly one caller moves
`Parked → Woken` and a second finds `Woken` already set and returns.

## Waking

`wake(coroutine, entry, epoch, result)` runs on the thread that owns the coroutine —
for every coroutine but one kind that is the thread it has always run on, and for
an actor that declared mid-message movement it is the thread its state word
names. The reactor drains its worker's completions there, a mailbox is read by
its actor's thread, and a signal from elsewhere reaches the reactor rather
than the coroutine. It does five things in order, behind one dispatch.

0. **Dispatch on the state word**, once, before anything else reads the
   record, with four outcomes (`dev/DECISIONS.md`, 2026-08-13). The word
   names **this thread**: continue into step 1. It names **another worker**:
   forward the payload to that worker's intake queue. It reads
   **`WokenShared`**: drop a wake and drop a conditional resolution, since
   the wait is already decided, but write the cancelled byte for a cancel —
   no worker owns the coroutine, and the one that wins the mount inherits the bit
   — and then re-read the word, because a single load orders nothing
   against the mount, so the answer follows the re-read
   (`design/cancellation.md`). It reads **`Terminal`**: drop the signal and
   release what it carried — releasing means *answering*, not discarding,
   for a cancel request, whose promise is resolved *already finished* here
   rather than broken by a dropped handle (`design/cancellation.md`).
   Dispatch stands ahead of the epoch check so that no foreign thread ever
   reads the epoch.

1. **Validate the epoch.** Compare the record's epoch with the argument.
   A mismatch means this entry was retired and the coroutine has since parked
   on something else, so the call returns. Without it a loser of a
   previous OR wait, firing between the win and the retirement, would
   wake the coroutine out of an unrelated wait with no entry fired at all.

   The coroutine itself cannot be gone: whoever armed the entry holds a
   counted reference to it, so it is alive for as long as the wake can
   arrive.
2. **Store the result** into that entry, and only into it.
3. **Decide whether the wait is over.** *Already decided* means the winner
   field is claimed or `remaining` has reached zero, and a caller that
   finds either returns here, before the counter is touched: a retired
   entry firing after `remaining` reached zero would otherwise drive it
   negative, and no reader is prepared for that. Step 2 stands ahead of
   this one because under AND a caller that does not reach zero returns
   here, and its result has to be stored before it does. Under **OR**, claim the
   winner field; a caller that finds it claimed returns. Under **AND**,
   decrement `remaining`; a caller that does not take it to zero returns.
4. **Retire the other entries** through their cancel handles. Retirement
   is asynchronous, so a retired entry may still fire. Once the coroutine has
   parked again, step 1 rejects it on the epoch; until then step 3 does,
   because the wait is already decided.
5. **Move the state word and enqueue**, and this is the one place a declared
   actor changes hands. Ordinarily: store `Woken`, or `WokenLocal(W)` for a
   declared actor, and enqueue into this worker's own list. On the move path,
   which needs the declaration, a foreign-frame counter of zero and this
   message's single move unspent: store-release `WokenShared`, then push to the
   ready set, then ring a sleeper. The word goes before the queue, and after
   that store this worker touches the coroutine no more — including its own later
   intake entries about it, which step 0 catches. A cancel always takes the
   ordinary path, because a coroutine that is unwinding has no reason to move.

Steps 1 through 4 are what make wakes safe to repeat. Step 5 is what
makes them arrive exactly once, and step 0 is what makes them safe to deliver
to a coroutine that has changed threads.

**`wake`'s answer is three-valued: accepted, refused, forwarded**, because a
caller holding the only news has to know where that news went. Refused, at step
0, 1 or 3 on this thread, means this coroutine carries it nowhere and a supply
resource must try the next waiter itself. Forwarded means the payload went to
another worker's intake queue and the obligation travelled with it, which is why
a supply wake's payload names the resource: the owner learns the refusal and
re-runs the wake there (`design/channels.md`). A resource that names an owner
has nothing to hand on and ignores the answer.

## The wait record

The wait record states what the coroutine is waiting for and who will end the
wait. It carries one entry per thing the coroutine waits on:

| Field | Meaning |
|---|---|
| resource | what is being waited on: a mutex, a channel, an actor, a join, an operation, a timer. A pooled one is named by its handle, everything else by a counted reference |
| cancel | an opaque handle that ends this entry early |
| result | where the waker stores what the coroutine will read |
| fired | whether this entry has already been satisfied |

**The record names a resource, not the coroutine that will end the wait.** What
the resource answers depends on its kind: a mutex or an actor names one
owner in a field, read fresh when anyone asks, while a channel or a future
names nobody and is answered by reachability (`design/deadlock.md`). Naming
a coroutine directly goes stale the moment a resource changes hands: two coroutines
parked on one mutex both record its holder, the holder releases it and
finishes, and the second waiter's record still names a coroutine that owns
nothing and will end no wait.

and, once per record: the mode, `remaining` for AND, the winner field, and
the `epoch` that every armed entry carries.

**The winner field exists in both modes**, because a wait can end without
any entry firing: a cancel ends one, and so does the detector's resolution
(`design/cancellation.md`, `design/deadlock.md`). Neither may reach that
end by writing `remaining`, because a counter another hand drove to zero is
indistinguishable from one the last entry drove there, and the entries are
still armed at that moment. Under OR the winner is the entry that fired
first. Under AND it stays unclaimed while the counter runs down, and
whoever does claim it has decided the wait outright.

**Beside the winner field the record carries one result slot of its own.**
A wait decided by no entry — a cancel, or the detector's resolution — stores
its error there and claims the winner with a reserved value no entry index
takes. An entry's result slot therefore keeps one writer, the wake that ends
that entry, which is why a retired entry firing late writes into storage
nobody reads. A resumed coroutine reads the winner field first: an entry index
points it at that entry's result, the reserved value at the record's own
slot, and an AND wait with the winner unclaimed means every entry is to be
read.

**Who can end a wait is answered by the resource, and how it answers
depends on its kind** (`design/deadlock.md`). A mutex names its holder, an
actor names the coroutine processing its current message, a join names its
target, an operation names the kernel: one debtor, in a field. A channel
and a future name nobody, because whoever holds the write end may serve
the wait, and the answer for them is which holders of that write end still
exist. `design/deadlock.md` reads all of this; nothing here does.

**The mode:**

- **AND** — the coroutine continues when every entry has fired. `remaining`
  starts at the number of entries.
- **OR** — the coroutine continues when any entry has fired, and the winner
  retires the others.

`await` with a timeout is the ordinary OR: a timer and an operation.

A wait edge has two ends. Recording what a coroutine waits on gives one of them,
and one end cannot close a cycle. `php-src/ext/async` records exactly that
end and nothing else (`dev/DECISIONS.md`, 2026-08-12). Here the second end
is answered by the resource, which is what keeps it true as ownership
moves. Neither end is a record field: an entry names the resource, and the
resource answers who can end the wait — a debtor by naming it, a channel
or a future by who can still reach its write end (`design/deadlock.md`).

**Writers and readers.** The record is written and read by the coroutine's own
thread and by nothing else: the coroutine writes it while `Running`, and
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

**Mounting a coroutine that changed threads takes the claim first.** Where the
state word reads `WokenShared`, the mounting worker compare-exchanges it to
`Running(W)` with acquire semantics, paired with the release store that put it
there, and installs the arena only after winning
(`dev/DECISIONS.md`, 2026-08-13). Whoever loses does not mount.

**Four accesses to a declared actor's word and byte are sequentially
consistent**: this compare-exchange, a canceller's store of the cancelled
byte, a canceller's re-read of the word, and park step 4's read of the byte.
Acquire and release alone leave the cancel unordered against the mount, so a
byte stored after step 4 has read it is never read again
(`design/cancellation.md`). For every other coroutine the byte stays a plain
store on one thread.

The hooks fire when a coroutine is mounted on a thread and when it leaves,
never per `poll` of a stackless coroutine. A stackless coroutine installs no actor
context, because actors are stackful, so the hot path of polling carries
no hook at all.

### Unmount takes a reason

The unmount hook receives which event it is, because the two have
opposite meaning for the collector and a hook that cannot tell them apart
will answer a handshake at the wrong moment.

**`Boundary`** — the coroutine is finished: its stack is empty, the actor's
state is consistent, and the collector's handshake can be answered
(`rfc/runtime/actors.md`). This is the safepoint the Limelight collector
depends on.

**`Park`** — the coroutine is suspended mid-message with live frames, some of
them foreign. The actor's state is whatever the message left it, so no
handshake may be answered and no collection may run over its arenas.

Treating a park as a safepoint would let the collector scan an actor
halfway through a message. Treating a boundary as a park would leave the
scheduler holding a coroutine with nothing left to run.

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
loops does not help, because a parked coroutine executes no loop. This is a
defect of the actor model rather than of the substrate, and it belongs in
`rfc/BACKLOG.md`, which does not yet carry it — the substrate is only
where it becomes visible.

## Thread-local storage, and who may move

**Only an actor ever moves between threads.** An ordinary coroutine shares its
thread's memory and is allocated in that thread's heap, so it is pinned
for life — a consequence of where its memory is, not a scheduling policy
(`dev/DECISIONS.md`, 2026-08-12). Everything in this section is therefore
about actors, and about them only.

**Our rule:** no thread-local address is cached across a suspension
point. A coroutine reaches its context through the coroutine, not through a
register that happened to hold a TLS address before the switch. After a
move that address belongs to the previous thread, and the arena it names
belongs to another actor.

**The rule has no enforcer today.** rustc offers no way to state "do not
keep this TLS address across this call", and whether LLVM in fact hoists
a `thread_local!` address across a suspension point is not measured here.
What the design does instead is remove the opportunity: the context is
reached through the coroutine on every path that can suspend, and a
suspendable path carries no callee-saved registers at all
(`design/switching.md`), so nothing survives a park in a register to
begin with. That argument has not been checked against generated code,
which is why it is a research item rather than a settled one.

**Foreign frames are outside the rule entirely.** A C library suspended
below our frame holds thread-local state we neither see nor move:
`errno`, an OpenSSL error queue, a locale handle. Nothing of ours is
broken when that state is read on the wrong thread after a move; it was
simply left behind.

**An actor may be re-mounted at any suspension point** (`dev/DECISIONS.md`,
2026-08-13). What differs between points is the price: between messages the
move carries nothing, because there is no coroutine and no stack, while inside a
message it carries the live stack and the saved registers, and it turns "the
coroutine's own thread" into a claim the state word names. That price is paid only
by an actor that declares mid-message movement, and such an actor moves at
most once per message. A non-zero foreign-frame counter vetoes a move at any
point, and a declaration cannot be combined with opting out of the counter:
the pair is rejected when the actor is created.

## Migration and the single `unsafe impl Send`

Moving an actor between threads is sound because at most one thread
executes it at any moment, and the scheduler handoff that moves it
carries the release and acquire pair that publishes its memory
(`rfc/runtime/actors.md`). Rust cannot check this: the compiler cannot
see a stackful coroutine's frames, so `corosensei` marks its own `Coroutine` type
`!Send` rather than guess.

The assertion lives in our code, in one place: the actor handle in the
scheduler carries a single `unsafe impl Send`, with the invariant above
written over it and the foreign-frame counter as its precondition. An
ordinary coroutine needs no assertion at all, because it never crosses.

A stackless coroutine written in Rust inside the substrate needs none either,
since rustc computes `Send` for its state machine the ordinary way. One
produced by the Limelight compiler is not covered by that: to rustc its
state is an opaque block behind size, alignment, `poll` and `drop`. Such
a coroutine crosses only when its producer declares that it may, and that
declaration is the second `unsafe` contract in the system.

## Restrictions on a stackless coroutine

A stackless coroutine may not host an actor, and may not suspend below a frame
it did not compile. Both come from the same fact: `poll` returns to its
caller, and there is no way to return `Pending` through a C frame that
expects a value.

Enforcement belongs to the compiler that produced the coroutine, and that is
why stackless coroutines exist only where we cooperate with one. Limelight
rejects such a body statically, because it sees both the suspension point
and the call that crosses the ABI. The substrate performs no runtime check
for this, because it cannot see inside an opaque `poll`.

Anything that needs to suspend below a foreign frame is a stackful coroutine.
That is the whole reason the stackful kind is the default.

## Lifecycle

```
                         ┌────────────────────────┐
                         ▼                        │
create ──▶ queued ──▶ Running ──┬──▶ Parked ──▶ Woken
                                │
                                └──▶ complete ──▶ Terminal ──▶ released
```

`Woken` returns to `Running` through the run queue. `Terminal` is teardown:
the state word says so from the moment the coroutine's own code has finished,
and `released` is the free that follows the last reference dropping.

There are three parking states and no fourth. A wake that arrives while the
coroutine is still suspending does not exist, because only the coroutine's own thread
touches it: the wake is delivered to that thread's intake queue and applied
after the coroutine is `Parked` (`dev/DECISIONS.md`, 2026-08-12). `Parked →
Woken` is enqueued by the waker, re-enters `Running` through the run
queue, and teardown is reached only from `Running`.

Creation allocates the coroutine object, which is an ordinary refcounted
entity of the memory manager, and for a stackful coroutine takes a stack from
the stack pool (`design/stacks.md`), then enqueues it. That is a third
enqueue, and it is not part of the wake invariant: the coroutine has no wait
record yet and no waker can name it.

Completion runs the consumer's unmount hook with reason `Boundary`, stores
`Terminal`, wakes whoever joined on it, releases the wait record, returns
the stack, and drops the scheduler's reference; the object is freed when the
last reference goes. The store sits where it does because everything the
coroutine's frames owned has been released by then and the record has not yet
been. The join wakes follow it rather than precede it, which is what lets
`design/deadlock.md` read a terminal target as live: a pass that sees
`Terminal` knows the wake is already on its way. The stack
needs no further condition because nothing the kernel may touch after
submission lives on it: buffers and submission structures come from the
buffer pool (`design/stacks.md`, `design/reactor.md`). Without that rule a
late completion would write into a stack already handed to another coroutine,
and a stack would have to outlive its owner.

### Cancellation moves the state word, not an entry

Cancelling a parked coroutine runs on the coroutine's own thread, the thread that
parked it, and a cancel raised elsewhere is posted to that thread's
reactor. On that thread the order is the wake protocol's, because a
cancellation is a wake carrying an error and an implementation has one code
path for both: store the cancellation in the record's own result slot;
decide the wait, which here means setting the cancelled bit and claiming
the winner with the reserved no-entry value, in either mode; retire the
entries through their cancel handles; store `Woken` and enqueue
(`design/cancellation.md`). Every store is plain, and the state store is
last for the same reason it is last in a wake. It does not go through
`wake`'s entry index and counter, because an AND wait's counter would
swallow it: a waker that does not take `remaining` to zero returns without
waking, and a cancelled coroutine would sleep forever waiting for entries that
were just retired.

**The claim is what makes the retired entries harmless, and the epoch is
not touched** — it is written once, in step 1 of the parking protocol
(`dev/DECISIONS.md`, 2026-08-13). An entry that fires after the claim finds
the wait decided and returns at step 3, whatever the mode; once the coroutine has
parked again, step 1 rejects it on the epoch instead. Bumping the epoch here
would give the record's identity a second writer and cover nothing the
decided test does not.

The bit is a modifier on the transitions that already exist, not a state
of its own. It is set by the coroutine's own thread, because a cancel raised
elsewhere reaches that thread's intake queue first.

### A coroutine always ends itself

There is no forced teardown. Ending a coroutine means making its current wait
finish with a cancelled result and letting it unwind from its own
suspension point; no thread unwinds another thread's frames, because the
frames below a suspension point may be foreign and their cleanup is not
ours to run.

The consequence for any policy that picks a victim — a deadlock victim
above all — is that a coroutine parked below a foreign frame that never returns
cannot be ended at all. The cancel request reports that outcome rather
than failing silently, which is what lets such a policy pick another
victim instead of repeating itself (`design/cancellation.md`).

## Decided elsewhere

All of these exist. The machinery an earlier version of this design used —
a coroutine named by a 64-bit handle with a slot and a generation, a
compare-and-swap on every parking transition, a seqlock over the wait
record — appears here and there as history and nowhere as a rule;
`dev/DECISIONS.md` of 2026-08-12 records what replaced it and why.

| Question | Document |
|---|---|
| what a context switch saves, and when it may save less | `design/switching.md` |
| stack reservation, commit, size classes, pooling | `design/stacks.md` |
| slot layout for pooled resources, and how pools are walked | `design/pool.md` |
| submitting operations, delivering completions, calling `wake` | `design/reactor.md` |
| retiring an entry, and releasing what the kernel holds | `design/cancellation.md` |
| proving a wait dead, and what is failed when it is | `design/deadlock.md` |

## Open questions

- **Mark termination waits on the slowest parked operation.** Stated
  above; owned by the actor model and not yet recorded in
  `rfc/BACKLOG.md`.
- **Backpressure on a parked actor's mailbox.** A coroutine parked mid-message
  blocks its actor, so messages accumulate behind it. `rfc/BACKLOG.md`
  carries the question; the substrate is where the queue grows.
- **Pinning granularity.** A coroutine stays pinned while a foreign frame is
  live and becomes migratable when it returns. Whether that is enough
  depends on what the library left in thread-local storage after
  returning, which we cannot inspect. The conservative alternative — pin
  for life once a foreign frame has been entered — costs the mid-message
  move for every declaring actor that ever calls out, so the narrower rule
  is what this document states and the one that needs testing against real
  libraries first.
- **Enforcing the TLS rule.** Stated above: no mechanism, only the absence
  of a reason to violate it, plus a migration test that does not exist
  yet.
