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
generation. A stale handle — a wake for an operation whose unit finished,
a cancel for a unit that completed — fails the comparison and does
nothing.

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

**Thirty-two bits wrap.** A slot recycled ten thousand times a second
wraps in about five days, and this design deliberately lets an operation
outlive the unit that submitted it, so a completion can be in flight for a
long time. Wrapping is survivable only because a stale completion must also
pass the epoch check of the wait its operation names; it is not proof, and a
64-bit generation with a narrower slot index is the fallback if the argument
turns out to be thin.

## Reclamation is deferred, and that is what makes waking safe

A completion resolves an operation's handle and then reads the slot: the
waiter it names, the entry index, the epoch to validate. Resolving and
reading are not one atomic step, so between them the operation may take its
last owed completion and its slot may be released.

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

## Walking a live pool

A walker iterates slabs and slots, reads each header with an acquire load,
and skips anything not in the state it wants. Its consumer is the diagnostic
dump that answers "what is everything waiting on", including the case no
detector can report: a write end held by live code that never writes
(`design/deadlock.md`). The detector itself walks nothing here — it computes
liveness over the collector's object graph — so the rules below serve the
dump alone.

**What a validated read guarantees, exactly:** that at some instant between
the two header loads, this slot held this occupant in this state. It
guarantees nothing before or after that instant, and nothing at all about
any other slot.

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
| target | the waiter's handle, the entry index, the epoch to validate |
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

A resource a unit can wait on answers who can end that wait, and there are
three kinds of answer (`design/deadlock.md`).

**An external resource is ended without any unit** — an operation by the
kernel, a timer by the wheel — so it holds no owner field and closes no
cycle.

**A debtor resource names its owner in a field:**

| Resource | Owner field |
|---|---|
| mutex | the unit holding it |
| semaphore, guard-released | its permit holders and its free count |
| actor, for a synchronous call | the unit processing its current message, or none |
| join | the unit being waited for |

The semaphore is a debtor only while its permits are released by dropping a
guard, which is what ties a release to an acquisition. One that anyone may
post without acquiring first — every semaphore reached over the C ABI —
belongs with the channels below (`design/deadlock.md`).

A wait record names the resource; the resource names its owner, read fresh
by whoever asks. That indirection is what keeps an edge true when a mutex
changes hands and the previous holder's slot is reused.

**A channel and a future name nobody.** Whoever holds the write end may
serve the wait, so no owner field can be right, and a registry of senders
would answer a question nobody asks: the detector needs who can still
reach the write end, which is reachability and not a field. What such a
resource does carry is its own state — buffer occupancy, the closed or
broken flag — and the rule that dropping the last write end closes it.

A consumer that builds a resource of its own over the C ABI takes on the
contract of its kind: a mutex keeps its owner field truthful, a channel
holds its write ends in the registered handle table, and a semaphore either
declares guard discipline and a truthful holder list or is treated as
postable and therefore always live. Nothing checks any of it, and the
detector's promise never to invent a deadlock rests on all three.

### Buffer, socket, timer

A buffer slot is a header, a length, the registered index if the pool is
registered, and the payload. A socket slot is a header, the descriptor or
its registered index, and the head and tail of the queue a multishot
operation appends to (`design/reactor.md`). A timer slot is a header, a
deadline, and the waiter's handle with its entry and epoch. None of them
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
