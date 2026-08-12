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
suspend there. Stackless units remain available for the cases in section
"Restrictions on a stackless unit", which exclude actors by construction.

## One handle, two kinds

The scheduler queues one handle type. A handle is a pointer to the unit
with the kind recorded in its low bits, which the unit's alignment leaves
free. The wake path never inspects the kind: it moves a handle to a run
queue. Dispatch reads it once, on mount, and branches to one of two
resume routines.

Everything downstream of that branch differs; everything upstream is
common. That is what keeps the two kinds from becoming two schedulers.

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

The substrate never sees the state machine's fields. It needs four things
from the compiler that produced it, and they form the whole contract:

| Item | Meaning |
|---|---|
| size, alignment | how much storage the unit's slot must provide |
| `poll(state, ctx) -> Ready(value) \| Pending` | resume, run until the next suspension or completion |
| `drop(state)` | release the state and everything it owns |
| `may_suspend_below_foreign` | always false; see the restrictions below |

`poll` is an ordinary call: it returns to its caller, which is the worker.
That is the whole difference from a stackful resume, and every other
difference follows from it.

### The worker stack is a bound, not a computation

A stackless unit runs on the worker's stack, so that stack must hold two
things at once: the deepest chain of ordinary calls between two
suspension points, and the depth of nested `poll` calls when futures are
composed, since an outer `poll` calls the inner one. Neither is derivable
from the source in general — recursion and foreign callbacks make it
unbounded — so the worker stack is guarded the way any thread stack is,
and overflow is a fault rather than a computed impossibility.

## One park primitive

Every suspension goes through one function. It writes the wait record and
returns a token saying whether the unit must actually suspend. What
happens next is the only thing that differs by kind:

- a stackful unit switches away and does not return until it is resumed;
- a stackless unit returns `Pending` and unwinds to whoever called `poll`.

The record is written **before** either, on the same thread, while the
unit is still running. Nothing else in the system may register a wait:
a wait recorded anywhere else is invisible to the deadlock detector, and
its absence is not detectable from the outside
(`dev/DECISIONS.md`, 2026-08-12).

## Parking states and the wake race

A completion can arrive on another thread while the unit is still
suspending. Between writing the wait record and finishing the switch, the
unit is running and parked at the same time, and mounting it there would
put two threads inside one actor — which is exactly what the non-atomic
refcounts of the Limelight actor model forbid (`rfc/runtime/actors.md`).

Four states, and the transition rules that resolve the race:

| State | Meaning | Who may leave it, and to where |
|---|---|---|
| `Running` | mounted on a thread | the unit itself → `Parking` |
| `Parking` | record written, suspension not finished | the unit → `Parked`; a waker → `Woken` |
| `Parked` | suspension finished, no thread holds it | a waker → `Woken` |
| `Woken` | a wake arrived; the unit is owed a run queue slot | whoever observes it → `Running` |

The rules that follow from the table:

- A waker that finds `Parking` sets `Woken` and enqueues nothing. The
  suspending side observes `Woken` when it tries to publish `Parked`,
  abandons the suspension and continues running.
- A waker that finds `Parked` sets `Woken` and enqueues the handle. It is
  the only path that enqueues, so a handle can never be in two queues.
- A second wake finding `Woken` does nothing. Wakes are idempotent
  because a wait may have several halves and more than one can fire.

`Parked` is published with a release store and read with an acquire load.
That ordering is what lets another thread — a waker, or the deadlock
detector — read a parked unit's wait record and see it whole.

## The wait record

The wait record states what the unit is waiting for and who will end the
wait. It carries one entry per half of the wait:

| Field | Meaning |
|---|---|
| resource | an opaque identifier of what is being waited on |
| poster | who will end this half: a unit, an actor, a set, or the kernel |
| cancel | an opaque handle that ends this half early |
| epoch | a counter bumped on every write, for readers that must validate |

`await` with a timeout is the ordinary case of more than one entry: a
timer and an operation, either of which may fire. The record therefore
states its mode:

- **AND** — the unit continues when every half has fired.
- **OR** — the unit continues when any half has fired, and the winner
  retires the others through their cancel handles.

Both modes exist because the deadlock detector treats them differently: a
cycle proves deadlock under AND, while under OR a single live half is
enough to keep everything moving (`design/deadlock.md`).

The `poster` field is what makes the record more than a debugging aid.
Recording only the resource gives half an edge, and half an edge cannot
close a cycle — the mistake this design exists to avoid.

Writers and readers: the parking unit writes the record while `Running`;
a waker reads the entry it is ending; the deadlock detector reads records
of units in `Parked` and writes nothing.

## Mount and unmount

Mounting installs the consumer's context and unmounting removes it. The
substrate does not know what that context is: the consumer supplies a
mount hook and an unmount hook, and Limelight's hook installs the actor
context and puts the current arena into thread-local storage. A consumer
without actors installs whatever it has, or nothing.

### Unmount at a boundary and unmount at a park

These are different events and only one of them is a safepoint.

**At a message boundary** the unit is finished: its stack is empty, the
actor's state is consistent, and the collector's handshake can be
answered (`rfc/runtime/actors.md`). This is the safepoint the Limelight
collector depends on.

**At a park** the unit is suspended mid-message with live frames, some of
them foreign. The actor's state is whatever the message left it, so no
handshake may be answered and no collection may run over its arenas.

Treating a park as a safepoint would let the collector scan an actor
halfway through a message. Treating a boundary as a park would leave the
scheduler holding a unit that has nothing left to run. The two paths
share only the unmount hook.

A consequence worth stating where it is first visible: an actor parked on
an operation that never completes never reaches a boundary, so the
collector's mark termination waits on it. Deadlock detection therefore
cannot be built on top of the collector finishing a cycle.

## Thread-local storage and pinning

**Our rule:** no thread-local address is cached across a suspension point.
A unit reaches its context through the unit, not through a register that
happened to hold a TLS address before the switch. After migration that
address belongs to the previous thread, and the arena it names belongs to
another actor.

**What the rule cannot cover:** foreign frames. A C library suspended
below our frame holds thread-local state we neither see nor move —
`errno`, an OpenSSL error queue, a locale handle. No rule of ours is
broken when that state is read on the wrong thread after migration; the
state was simply left behind.

**Therefore a unit with a live foreign frame is pinned** to the thread it
is running on, and stays pinned until that frame returns. The scheduler
does not steal a pinned unit. The bit that marks a live foreign frame is
the same one that decides how much a context switch must save
(`design/switching.md`), so it is set and cleared in one place and read
in two.

## Migration and the single `unsafe impl Send`

Migration is sound here because at most one thread executes a unit at any
moment, and the scheduler handoff that moves it carries the release and
acquire pair that publishes its memory (`rfc/runtime/actors.md`). Rust
cannot check this: the compiler cannot see a stackful unit's frames, so
`corosensei` marks its coroutine `!Send` rather than guess.

The assertion therefore lives in our code, in one place: the handle type
in the scheduler carries a single `unsafe impl Send`, with the invariant
above written above it and the pinning rule as its precondition. No other
`unsafe impl Send` is added for units. Stackless units need none, because
the compiler computes `Send` for a state machine the ordinary way.

## Restrictions on a stackless unit

A stackless unit may not host an actor, and may not suspend below a frame
it did not compile. Both restrictions come from the same fact: `poll`
returns to its caller, and there is no way to return "pending" through a
C frame that expects a value.

Enforcement follows the compiler that produced the unit. Limelight
rejects such a body statically, because it sees the suspension point and
the call that crosses the ABI. A stackless unit registered over the C ABI
carries `may_suspend_below_foreign = false`, and a park attempt below a
foreign frame is a runtime error rather than a corrupted stack.

Anything that needs to suspend below a foreign frame is a stackful unit.
That is the whole reason the stackful kind is the default.

## Lifecycle

```
create ──▶ Running ──▶ Parking ──▶ Parked ──▶ Woken ──▶ Running ──▶ complete
             │                                                          │
             └──────────────── complete without parking ────────────────┘
```

Creation takes a slot from the unit pool (`design/pool.md`) and, for a
stackful unit, a stack from the stack pool (`design/stacks.md`).
Completion runs the consumer's unmount hook, releases the wait record,
and returns the slot. The stack returns to its pool only after every
operation that named it has confirmed that no kernel reference into it
remains (`design/cancellation.md`).

### Cooperative completion and forced teardown

A unit normally ends by finishing its body. Forced teardown — ending a
unit that has not agreed to end — is available only where the frames can
be unwound: unwinding through a foreign frame is undefined unless that
frame was compiled to allow it, and the cleanup it would skip is
arbitrary.

A unit with a live foreign frame is therefore not force-killable, only
cancellable at its next suspension point. Any policy that picks a victim
— a deadlock victim above all — inherits this limit and must state what
it does when the victim it wants is unkillable.

## Decided elsewhere

These documents are not yet written; the plan lists them in order
(`dev/PLAN.md`).

| Question | Document |
|---|---|
| what a context switch saves, and when it may save less | `design/switching.md` |
| stack reservation, commit, size classes, pooling | `design/stacks.md` |
| the layout of a unit slot and how pools are walked | `design/pool.md` |
| submitting operations and delivering completions | `design/reactor.md` |
| cancelling a wait and releasing what the kernel holds | `design/cancellation.md` |
| finding wait cycles, and the victim policy | `design/deadlock.md` |

## Open questions

- **Backpressure on a parked actor's mailbox.** A unit parked mid-message
  blocks its actor, so messages accumulate behind it. The bound belongs to
  the actor model rather than here, and `rfc/BACKLOG.md` already carries
  it, but the substrate is where the queue actually grows.
- **Pinning granularity.** A live foreign frame pins a unit to its thread.
  Whether a unit that has ever held a foreign frame may migrate again
  after that frame returns depends on what the library left in
  thread-local storage, which we cannot inspect. The conservative rule —
  pin only while the frame is live — is what this document states, and it
  is the one that needs testing against real libraries first.
