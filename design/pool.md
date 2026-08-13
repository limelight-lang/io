# Object Pools

## What a pool is

A pool is a set of slabs holding fixed-size slots of one kind: sockets,
timers, buffers, and the operations in flight against them. A coroutine and
an actor are not pooled. A coroutine is an object of the memory manager and
its lifetime is its reference count; an actor's header lives in the arena it
owns, so that it travels with the memory it works with
(`dev/DECISIONS.md`, 2026-08-12 and 2026-08-13).

Pools exist for three reasons, and only the first is about allocation:

- **Submitting an operation allocates nothing.** Its slot holds what the
  kernel's completion will need, so submission is a write rather than a
  malloc on a path that runs once per operation. Parking allocates nothing
  either, for a reason that is not the pool's: the waker is embedded in the
  coroutine (`dev/DECISIONS.md`, 2026-08-12).
- **Enumeration needs no registry.** Walking the slabs of a pool lists
  every object of that kind. `php-src/ext/async` keeps a hash table of
  coroutines and another of channels in a potential-deadlock state; the
  same information here is the pool itself, and it cannot drift from
  reality because it is not a copy of it.
- **The kernel can be told about buffers once.** A registered buffer pool
  is pinned once and referenced by index afterwards, which is only
  possible because slabs do not move (`design/reactor.md`).

## The handle

**A handle is a 64-bit word, not a pointer.**

| Bits | Field |
|---|---|
| 63–60 | pool |
| 59–32 | slot index |
| 31–0 | generation |

Resolving a handle checks the slot index against the pool's current
extent, computes the slot's address from the slab table, and compares the
generation. A stale handle — a completion for an operation whose slot was
released and handed out again, a cancel naming a timer that already
fired — fails the comparison and does nothing.

**A handle names a pooled object and nothing else.** A coroutine and an
actor are named by counted references: whoever arms an entry of a wait holds
one, so `wake` cannot find its target missing and needs no generation to
check (`design/execution.md`). What travels as a handle is what a pool owns
— an operation, a buffer, a socket, a timer — and it travels across thread
boundaries, where a stale one must fail rather than resolve.

A generation does not fit beside a pointer in one word, which is why a
pooled object is named by a handle rather than by a pointer with flags.

## Slabs do not move and are not freed

A slab is one mapping, allocated from the consumer's memory manager, and
it stays mapped for the life of the process. Registered buffers hold
kernel references to fixed addresses, and a walker reading a slot must
not have the slab disappear under it.

Growth appends slabs and never reallocates. The slab table is
append-only, published with a release store and read with an acquire
load, so a walker that started before a slab was added does not see it.

## The slot header

| Field | Width | Meaning |
|---|---|---|
| `state` | one word, atomic | free, or the kind's own live states |
| `generation` | 32 bits, atomic | incremented once, at allocation |
| `kind` | 8 bits | which pool this slab belongs to; read for assertions, and by the detector to pick a resource's liveness rule (`design/deadlock.md`) |

The generation is incremented **when a slot is allocated**, not when it is
released, and it is the only counter that distinguishes one occupant of a
slot from the next. A coroutine's wait epoch is a different counter in a
different place: it distinguishes one wait of one coroutine
(`design/execution.md`), and nothing in a pool reads it.

**Thirty-two bits wrap, and wrapping is harmless, because a comparison only
ever spans a bounded window.** A slot still owed completions, or held
through a pending cancel, does not release at all, so its generation is
frozen while its handle is legitimately in use. After a release, each reuse
requires every worker — the eventual reader included — to pass a quiescent
point, and every comparison the substrate performs happens at the drain of
the reading worker's current or next turn: an undrained completion this
turn, a posted retire or cancel the next, an inline use at once. The window
therefore admits a few hundred reuses of one slot where a false pass needs
exactly 2³².

**This holds because no pool handle crosses the API**, and that is now a
rule rather than an accident: the reactor's calls take descriptors, and the
C-ABI surface hands out counted objects and descriptors and no pool name. A
consumer-retained handle would have an unbounded staleness window and would
alias in days at the recycle rate above. A surface that hands one out
reopens the width question for that pool alone, and a wider generation
behind that pool's nibble is the answer there rather than a global widening.

The wait epoch is not part of this argument and never was. It tells one
wait of one coroutine from the previous one, and the completion carries no
epoch of its own to check — the epoch it validates comes out of the slot's
own waiter cell.

## Reclamation is deferred, and that is what makes waking safe

A completion resolves an operation's handle and then reads the slot: the
owed count, the buffer it pins, the waiter cell. Resolving and reading are
not one atomic step, so between them the operation may take its last owed
completion and its slot may be released.

**A released slot is not handed out again until every worker has passed a
quiescent point after the release.** Workers are quiescent between
messages, which they reach constantly, so the delay costs a per-worker
counter published once per scheduler turn and nothing on the completion
path.

What this buys: a late reader reads a slot that is free but not yet reused,
sees the free state and stops. A slot handed out immediately would instead
give it a live occupant's fields, and a completion would be attributed to an
operation that never ran.

The generation is what rejects a read that arrives *after* reuse; the
deferred reclamation is what rejects one that arrives *during* it. Both
are needed, and the earlier design had only the first.

**Both serve handle resolution and the dump, and neither serves the wake
path**, which no longer needs them: a waiter cell has one thread, so a
completion never loads a reference another thread is dropping.

## Walking a live pool

A walker iterates slabs and slots, reads each header with an acquire load,
and skips anything not in the state it wants. Its consumer is the diagnostic
dump that answers "what is everything waiting on", including the case no
detector can report: a write end held by live code that never writes
(`design/deadlock.md`). The detector itself walks nothing here — it computes
liveness over the collector's object graph — so the rules below serve the
dump alone.

**The collector reads nothing in a pool either.** An armed cell's unit is
reachable through the scheduler's ownership table while its wait is
undecided, and through the intake entry of a pending cross-thread retire
afterwards; both are roots the collector already has, so no scan of slabs
covers a cell (`design/deadlock.md`, `dev/DECISIONS.md`, 2026-08-13). A
scan that did would be this walk, whose first rule is that it may miss a
slot.

**What a validated read guarantees, exactly:** that at some instant between
the two header loads, this slot held this occupant in this state. It
guarantees nothing before or after that instant, and nothing at all about
any other slot. **The dump never follows a waiter cell**: a validated read
licenses the header and nothing behind it, and the reference in the cell
belongs to the slot's own thread.

Two rules follow:

1. **Accept false negatives.** A slot that changes state after the walker
   passed it is reported as it was, and the walk runs again.
2. **A conclusion spanning several slots must be re-validated as a
   whole.** Slots are read at different instants, so a set assembled from
   them was never observed simultaneously. A dump that draws a wait graph
   from several slots is describing a system that may never have existed in
   that state, and it says so rather than pretending otherwise.

The cost is a linear read of live slots in address order, with no pointer
chasing.

## Slot layouts

### Where a coroutine and an actor live instead

Neither is a slot, and this is the largest of the corrections this document
carries. A coroutine is an object of the memory manager: its wait record and
two inline entries sit in the object, further entries spill into one raw
block freed through `deferred_free`, and the scheduler holds one counted
reference for as long as it owns the coroutine (`dev/DECISIONS.md`,
2026-08-12). An actor's header sits in the arena the actor owns, so that it
travels with the memory it describes: the state word, the mailbox, the unit
mounted for the message in flight, and the intrusive link by which the
scheduler queues it (`dev/DECISIONS.md`, 2026-08-13). The saved machine
context is in neither place — it sits at the top of the coroutine's own
stack (`design/switching.md`), which is a pooled object in its own right.

What the detector asks about an actor is therefore read from that header
rather than from a slot: whether a unit is mounted, whether one is queued,
and whether the mailbox is empty (`design/deadlock.md`). An actor that is
idle with an empty mailbox is answered by the liveness table of that
document and not here.

### Operation

| Group | Contents |
|---|---|
| header | `state`: submitted, result received, awaiting notification, cancelling |
| waiter cell | a counted reference to the waiting unit, the entry index, the epoch to validate — or empty |
| memory | the buffer handle it pins, and the buffer's registered index if it has one |
| accounting | how many completions the kernel still owes |

**The waiter cell has one writer, one emptier and one thread**, and that
thread is the slot's owner. For an operation and a timer it is the unit's
own thread, because arming submits to this worker's own ring and wheel
(`dev/DECISIONS.md`, 2026-08-13), so the cell is written at step 2 of the
parking protocol. The same worker empties it, at whichever of two events
comes first. The completion carrying the operation's result
takes the reference out of the cell and moves it into the `wake` it calls,
and whoever consumes that payload drops it, exactly as a channel node's
reference moves (`design/channels.md`). Retiring the entry empties the cell before the
kernel cancel is submitted. Applied inline when the wait ended on this
worker, it drops the reference there and then. **A cross-thread retire drops
nothing where it runs**: the request carries a counted reference of its own,
taken by the poster on the unit's thread, and the applier moves the cell's
reference into the confirmation it posts back, so both come home to be
dropped where the count belongs. No thread ever decrements a count it does
not own, which is the shape the completion path already has. A retire that
finds the result already received submits no kernel cancel, there being
nothing left to hasten.

Two slots hold no waiter cell of their own. **A multishot operation names
the socket's handle and never a unit**, because the unit re-parks between
chunks (`design/reactor.md`); the reference for a wait on that queue sits
in the socket slot's queue-waiter cell, which is a cell of this kind. **A
cancel operation** names the operation it cancels and no waiter at all.

**An operation is not released on its first completion**, because the
kernel does not always owe exactly one:

- a zero-copy send owes two, the result and the notification that
  releases the buffer (`dev/DECISIONS.md`, 2026-08-12);
- a multishot operation owes many, ending with a completion that does not
  say more are coming;
- a cancelled operation owes its own completion in addition to the
  cancel's, and the cancel is a separate operation with a separate slot
  (`design/cancellation.md`).

The slot is released, and its buffer unpinned, when the owed count reaches
zero and not before. **A completion arriving at an empty waiter cell calls
no `wake`**: it adjusts the owed count, applies its buffer rule, and
releases the slot if the count reached zero. Slot release involves no unit,
which is what keeps an operation abandoned against a hung mount from
outliving anything but its buffer and its slot (`design/cancellation.md`).

### Where a waitable resource lives instead

Three of the things a wait entry can name are pool slots, and all three are
the external kind: an operation, ended by the kernel; a timer, ended by the
wheel; and a socket, whose completion queue a multishot operation appends
to (`design/reactor.md`, `design/deadlock.md`). None holds an owner field,
because none of those waits is ended by a coroutine, and the `kind` byte of
the slot is what the detector reads to pick the rule.

Every other resource a unit can wait on is an entity of the memory manager
or of the thread's heap, and its kind is read from its class rather than
from a slot header. A mutex, a join, a synchronous actor call and a
semaphore whose permits are released only by dropping a guard name one
owner and are debtors; a channel, a future, an actor's mailbox and a
semaphore anyone may post name nobody, because whoever holds the write end
may serve the wait. Which fields each exposes, and how a consumer that
builds one over the C ABI keeps its answers truthful, belong to
`design/deadlock.md` and to `design/channels.md`; nothing about them is
decided here.

### Buffer, socket, timer

A buffer slot is a header, a length, the registered index if the pool is
registered, and the payload. A socket slot is a header, the descriptor or
its registered index, and the head and tail of the queue a multishot
operation appends to, plus the queue-waiter cell a unit parking on that
queue writes (`design/reactor.md`). A timer slot is a header, a deadline
and a waiter cell. All three cells obey the same invariant, and it is *one
thread* rather than *the waiting unit's thread*: the cell belongs to the
worker that owns the slot, which for an operation and a timer is the
worker that submitted or armed it, and for a socket's queue-waiter cell is
the worker whose ring carries the multishot. A unit parked from any other
worker reaches the cell through that worker's intake queue, and the
park-time recheck of the queue is performed by whichever thread writes the
cell, immediately after the write (`design/reactor.md`). None of these slots carries a wait
record: only units wait.

## Allocation and release

Each worker holds a small free list per pool, refilled from and returned
to a shared free list in batches.

- **Take:** pop from the worker's list; on empty take a batch from the
  shared list; on empty append a slab. The pop acquires, pairing with the
  release below. The slot is initialized and its generation incremented
  **before** its first live state is published, so a walker never sees a
  live state over an uninitialized record.
- **Release:** store `free` into the state word, then publish the slot to
  the retire list of the current epoch. It becomes takeable only after
  every worker has passed a quiescent point.
- **Batch return** keeps a worker finishing many units off the shared
  list.

## Registered buffers and their limits

Registration is what makes contract 2 in `design/reactor.md` worth having,
and it constrains this design in ways worth stating here rather than
discovering in the backend.

- **Growth needs a sparse registration.** Appending a slab after
  registration leaves its buffers outside the registered table. The fixed
  table is replaced only by unregistering and registering again, which
  quiesces the ring; a sparse registration updated in place avoids that
  and sets a kernel floor the project has to declare. Which floor, and
  whether the fallback is per-slab rather than per-process, is open.
- **A registered slab's pages may never be dropped.** Registration pins
  physical pages; dropping the user mapping's pages replaces them while
  the kernel keeps the originals, so a later fixed read succeeds into
  pages the process can no longer see. This is a rule, not a tuning
  knob, and it is why the stack pool's release protocol
  (`design/stacks.md`) has no counterpart here.
- **Pinned pages are charged** against the process's locked-memory limit,
  so a container with a small default fails registration at startup.
- **The index space is shared and finite.** Buffer indices are a single
  per-ring namespace, so several buffer size classes must partition one
  index range between them. No document owns that partition yet.

## The memory manager underneath

Slabs come from the consumer's block allocator — for Limelight, the same
one that serves arenas. The substrate implements no page allocator and
does not use the system allocator for slabs, because registered buffers
need stable, known addresses.

The contract it needs is small: a page-aligned block of a requested size,
mapped until the process ends, registrable with the kernel. A consumer
that cannot promise stability gets an unregistered buffer pool and loses
the fixed-buffer path, which is a performance loss and not a correctness
one.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record's meaning, the park order, `wake` | `design/execution.md` |
| stacks, which are not pool slots | `design/stacks.md` |
| buffer contracts and registration | `design/reactor.md` |
| when an operation stops being owed completions | `design/cancellation.md` |
| what a walk concludes, set-level re-validation, the three resource kinds | `design/deadlock.md` |
| channels, futures, and what each exposes to the detector | `design/channels.md` |

## Open questions

- **Slab size**, against the mapping ceiling measured in
  `design/stacks.md`. No measurement.
- **The kernel floor for sparse buffer registration**, and whether the
  fallback for a grown pool is per-slab registration.
- **Partitioning the buffer index namespace** across size classes.
- **Reclaiming slabs.** Never freeing is right for a server and wrong for
  a process that peaks once. Deregistration before unmapping is a global
  operation on some backends, which is what makes it hard.
- **Whether sockets and timers want one pool or two.** They differ in size
  and in walk frequency; cheap to change before there is code.
