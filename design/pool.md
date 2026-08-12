# Object Pools

## What a pool is

A pool is a set of slabs holding fixed-size slots of one kind. Every
object the substrate owns lives in one: units, actors, sockets, timers,
buffers, and the operations in flight against them.

Pools exist for three reasons, and only the first is about allocation:

- **Parking allocates nothing.** A unit's wait record lives in its slot,
  so recording a wait is a write rather than a malloc on the path that
  runs once per suspension.
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
generation. A stale handle — a wake for an operation whose unit finished,
a cancel for a unit that completed — fails the comparison and does
nothing.

Everything that names an object across a thread boundary carries a
handle: run queue entries, wakers, cancel handles, and the `poster` field
of a wait record. `wake` carries one too
(`design/execution.md`), which is what lets a waker validate the occupant
and not merely the wait.

A pointer with flags in its low bits was the earlier design and is gone.
The suspension kind lives in the slot, which dispatch reads anyway, and
a generation does not fit beside a pointer in one word.

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
| `kind` | 8 bits | which pool this slab belongs to, for assertions |

The generation is incremented **when a slot is allocated**, not when it is
released, and it is the only counter that distinguishes occupants. The
epoch inside a unit's wait record distinguishes waits of one occupant.
A wake validates both, and it can, because it carries the handle.

**Thirty-two bits wrap.** A slot recycled ten thousand times a second
wraps in about five days, and this design deliberately lets an operation
outlive the unit that submitted it, so a wake can be in flight for a long
time. Wrapping is survivable only because a stale wake must also match
the epoch of a wait that no longer exists; it is not proof, and a 64-bit
generation with a narrower slot index is the fallback if the argument
turns out to be thin.

## Reclamation is deferred, and that is what makes waking safe

A waker validates a handle and then writes: the result into a half's
slot, then a counter, then the state word (`design/execution.md`).
Validating and writing are not one atomic step, so between them the unit
may complete and its slot may be released.

**A released slot is not handed out again until every worker has passed a
quiescent point after the release.** Workers are quiescent between
messages, which they reach constantly, so the delay costs a per-worker
counter published once per scheduler turn and nothing on the wake path.

What this buys: a late waker writes into a slot that is free but not yet
reused, which harms nothing, and then fails its final compare-and-swap on
the state word because the state is free. A slot handed out immediately
would instead take those writes into a live occupant's record — a result
in a half that never fired, a decremented counter, a unit woken out of a
wait nothing satisfied.

The generation is what rejects a wake that arrives *after* reuse; the
deferred reclamation is what rejects one that arrives *during* it. Both
are needed, and the earlier design had only the first.

## The wait record is a seqlock

A walker reads a parked unit's record while other threads may write it.
The epoch doubles as the sequence number that makes the read consistent:

- **Odd means being written.** A unit entering a park makes the epoch odd,
  writes the entries, then makes it even. `design/execution.md` orders
  parking as state, then record, then arm, and this is the record step in
  detail.
- **A reader** loads the epoch, reads the entries, then re-loads the
  epoch. Equal and even means the entries belong to one wait. Anything
  else means retry or abandon.

Without this the walker reads the entries of a new wait under the epoch of
the old one, because the epoch was written first, and composes a wait that
never existed.

## Walking a live pool

A walker iterates slabs and slots, reads each header with an acquire load,
and skips anything not in the state it wants. Its consumer is the
diagnostic dump that answers "what is everything waiting on". The deadlock
detector does **not** walk the pool: it follows resource handles from one
slot to the next, which costs the reachable subgraph rather than the
population (`design/deadlock.md`). The rules below serve both, because
both read slots other threads are writing.

**What a validated read guarantees, exactly:** that at some instant
between the two epoch loads, this slot held this occupant, parked on this
wait. It guarantees nothing before or after that instant, and nothing at
all about any other slot.

Three rules follow, and the third is the one the earlier version was
missing:

1. **Re-read the state word, not only the generation and the epoch.** A
   unit's epoch changes when it *parks again*, not when it wakes, so a
   unit that woke and is running still carries the epoch the walker first
   saw. Only the state word says it is no longer parked.
2. **Accept false negatives.** A unit that parks after the walker passed
   it is missed, and the walk runs again.
3. **A conclusion spanning several slots must be re-validated as a
   whole.** Slots are validated at different instants, so a set assembled
   from them was never observed simultaneously. `design/deadlock.md`
   re-validates every member after its candidate set closes, and this
   sentence is the contract it relies on.

The cost is a linear read of live slots in address order, with no pointer
chasing.

## Slot layouts

### Unit

| Group | Contents |
|---|---|
| header | `state`, `generation`, `kind` |
| suspension | kind (stackful or stackless), stack pointer for the former, state-machine pointer and its vtable for the latter |
| wait | mode, `epoch`, `remaining`, winner, the detector's claim word, and two entries: resource handle, cancel handle, result, fired |
| scheduling | run queue link, the foreign-frame counter, the affinity flag, the retryable flag the consumer sets at creation |
| consumer | one opaque word the mount hook owns, where Limelight keeps its actor context |

The saved machine context is not here: it sits at the top of the unit's
own stack (`design/switching.md`), so the slot's size does not follow the
platform's register file.

Two wait entries cover an operation and a timer, which is `await` with a
timeout. A wait with more halves takes a spill slot from the same pool
and stores its handle in place of the entries. A spill slot's state marks
it as a spill so a walker skips it as a unit, and it is freed under the
same deferred reclamation as any slot, so a walker mid-read is safe.

### Actor

| Group | Contents |
|---|---|
| header | `state`: idle, has messages, or running |
| current | the handle of the unit mounted for the message in flight, or none |
| mailbox | head and tail, and the queued count |

**Actors need slots because the wait edge points at them.** A unit parked
on a reply from another actor names that actor, and the walker must be
able to ask what the actor is doing. It follows `current` to the unit
processing the actor's message and continues from there, which is how an
actor-mediated cycle closes — the primary case for a PHP runtime.

An actor that is idle with an empty mailbox has no unit to follow. The
walker treats such an edge as live rather than blocked, because any
sender could wake it and the walker cannot prove none will. That is a
false negative by construction and it is the safe direction.

### Operation

| Group | Contents |
|---|---|
| header | `state`: submitted, result received, awaiting notification, cancelling |
| target | the waiter's handle, the half index, the epoch to validate |
| memory | the buffer handle it pins, and the buffer's registered index if it has one |
| accounting | how many completions the kernel still owes |

**An operation is not released on its first completion**, because the
kernel does not always owe exactly one:

- a zero-copy send owes two, the result and the notification that
  releases the buffer (`dev/DECISIONS.md`, 2026-08-12);
- a multishot operation owes many, ending with a completion that does not
  say more are coming;
- a cancelled operation owes its own completion in addition to the
  cancel's, and the cancel is a separate operation with a separate slot
  (`design/cancellation.md`).

The slot is released, and its buffer unpinned, when the owed count
reaches zero and not before.

### Resource

Anything a unit can wait on has a slot, and every such slot answers one
question: who owes this wait right now.

| Resource | Owner field |
|---|---|
| mutex, semaphore | the unit holding it |
| actor | the unit processing its current message, or none |
| channel | its registered senders, or its registered receivers |
| operation | the kernel |
| timer | the wheel |

A wait record names the resource; the resource names its owner, read fresh
by whoever asks. That indirection is what keeps an edge true when a mutex
changes hands and the previous holder's slot is reused
(`design/deadlock.md`).

A consumer that builds a mutex or a channel of its own over the C ABI
takes on keeping its owner field truthful. Nothing checks it, and the
detector's promise never to invent a deadlock rests on it.

### Buffer, socket, timer

A buffer slot is a header, a length, the registered index if the pool is
registered, and the payload. A socket slot is a header, the descriptor or
its registered index, and the head and tail of the queue a multishot
operation appends to (`design/reactor.md`). A timer slot is a header, a
deadline, and the waiter's handle with its half and epoch. None of them
carries a wait record: only units wait.

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
| what a walk concludes, and set-level re-validation | `design/deadlock.md` |

## Open questions

- **Slab size**, against the mapping ceiling measured in
  `design/stacks.md`. No measurement.
- **The kernel floor for sparse buffer registration**, and whether the
  fallback for a grown pool is per-slab registration.
- **Partitioning the buffer index namespace** across size classes.
- **Generation width.** Thirty-two bits wrap in days at a high recycle
  rate, and the argument that the epoch covers the wrap is not a proof.
- **Reclaiming slabs.** Never freeing is right for a server and wrong for
  a process that peaks once. Deregistration before unmapping is a global
  operation on some backends, which is what makes it hard.
- **Whether sockets and timers want one pool or two.** They differ in size
  and in walk frequency; cheap to change before there is code.
