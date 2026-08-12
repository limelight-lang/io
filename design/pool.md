# Object Pools

## What a pool is

A pool is a set of slabs holding fixed-size slots of one kind. Every
object the substrate owns lives in one: units, sockets, timers, buffers,
and the operations in flight against them. A slot is addressed by a slab
index and a slot index, and a handle carries both plus a generation.

Pools exist for three reasons, and only the first is about allocation:

- **Parking allocates nothing.** A unit's wait record lives in its slot,
  so recording a wait is a write rather than a malloc on the path that
  runs once per suspension.
- **Enumeration needs no registry.** Walking the slabs of a pool lists
  every object of that kind. `php-src/ext/async` keeps a hash table of
  coroutines and another of channels in a potential-deadlock state; the
  same information here is the pool itself, and it cannot drift from
  reality because it is not a copy of it.
- **The kernel can be told about them once.** A buffer pool registered
  with `IORING_REGISTER_BUFFERS` is pinned once and referenced by index
  afterwards, which is only possible because the slabs do not move
  (`design/reactor.md`).

## Slabs do not move and are not freed

A slab is one mapping, allocated from the consumer's memory manager, and
it stays mapped for the life of the process. Two things depend on that
and neither has an alternative: registered buffers hold kernel references
to fixed addresses, and a walker reading a slot must not have the slab
disappear under it.

Growth adds slabs; it never reallocates. The slab list is append-only,
published with a release store, and read with an acquire load, so a
walker that started before a slab was added simply does not see it.

## The slot header

Every slot of every kind begins with the same three fields, because the
walker and the handle machinery are shared across kinds.

| Field | Width | Meaning |
|---|---|---|
| `state` | one word, atomic | free, or the kind's own live states |
| `generation` | 32 bits, atomic | incremented every time the slot is allocated |
| `kind` | 8 bits | which pool this slab belongs to, for assertions |

**The generation is what makes a handle safe.** A handle names a slot and
carries the generation it was taken at. A stale handle — a wake for an
operation that finished, a cancel for a unit that completed — finds the
generation changed and does nothing. Without it, a slot reused by an
unrelated object would accept the stale operation, which is the same
class of bug the epoch check prevents inside a single unit's life
(`design/execution.md`).

Generation and epoch are two counters at two scales: the generation
distinguishes occupants of a slot, the epoch distinguishes waits of one
occupant. A wake validates both.

## The unit slot

| Group | Contents |
|---|---|
| header | `state`, `generation`, `kind` |
| suspension | kind bit (stackful or stackless), stack pointer for the former, state-machine pointer and its vtable for the latter |
| wait | the wait record: mode, `epoch`, `remaining`, winner, and the entries with resource, poster, cancel handle and result slot |
| scheduling | run queue links, the foreign-frame counter, the affinity flag |
| consumer | one opaque word the mount hook owns, which is where Limelight keeps its actor context |

**The saved machine context is not here.** It sits at the top of the
unit's own stack (`design/switching.md`), so the slot size does not depend
on the platform's register file and does not change if the switch does.

**Alignment is at least four bytes**, because a handle records the
suspension kind in the pointer's low bits and one spare bit is kept
(`design/execution.md`). The slot layout therefore has an alignment
requirement it did not choose, and this is where it is written down.

The wait record has a fixed number of entries. Two is the working number:
an operation and a timer covers `await` with a timeout, which is the
common multi-half wait. A wait with more halves spills its entries into a
side slot from the same pool, and the record holds a pointer instead —
paid by the rare case rather than by every slot.

## Walking a live pool

A walker iterates slabs and slots, reads each header with an acquire
load, and skips anything that is not in the state it wants. It writes
nothing. The deadlock detector is the first consumer, and a diagnostic
dump is the second (`design/deadlock.md`).

The system runs while the walk proceeds, so the walk sees a smear rather
than a snapshot. What it may and may not conclude:

- **A slot read as `Parked` may already be running.** The walker's
  conclusion is therefore never acted on from the read alone: it
  re-reads the generation and the epoch after collecting a slot's fields
  and discards the slot if either changed.
- **A unit that parks after the walker passed it is missed.** That is a
  false negative and it is acceptable: the walk runs again, and a real
  deadlock does not go away.
- **A phantom is not acceptable.** Two halves read from two different
  waits of the same slot would compose into a wait that never existed,
  which is what the epoch re-read prevents.

The cost is a linear read of the pool's live slots, in address order,
with no pointer chasing. That is the property the pools were chosen for:
the alternative — a registry of parked units — costs a write on every
park and can disagree with the truth.

## Allocation and release

Each worker holds a small free list per pool, refilled from and returned
to a shared free list in batches. The path that matters is one message
long: take a unit slot, take a stack, run, return both.

- **Take:** pop from the worker's free list; on empty, take a batch from
  the shared list; on empty, append a slab.
- **Release:** increment the generation, then publish the slot to the
  worker's free list. That order matters for the same reason release
  ordering matters for stacks (`design/stacks.md`): a slot visible on a
  free list with a stale generation can be taken and then invalidated by
  the increment.
- **Batch return** keeps a worker that finishes many units from touching
  the shared list on every one.

A slot is never returned while any handle to it may still be dereferenced.
For units that is guaranteed by the state machine: a slot is released
from `Running` at completion, when no waker holds a claim. For sockets and
buffers it is guaranteed by the operation count in the next section.

## Operations, and what they pin

An operation slot is taken when an operation is submitted and released
when its completion arrives — including the completion of a cancel, which
is a completion like any other (`design/cancellation.md`). It records
which buffer it uses, which unit to wake, and the epoch and generation to
validate against.

While an operation is live it pins its buffer: the buffer slot is not
released, and its pages are not dropped. This is where the rule that
nothing kernel-visible lives on a stack has its counterpart — the memory
the kernel writes into is pool memory, and the pool knows how long it is
owed (`dev/DECISIONS.md`, 2026-08-12).

A unit does not pin anything. It can complete with operations still in
flight; the operations hold what they need, and their wakes find a
changed generation and do nothing.

## The memory manager underneath

Slabs come from the consumer's block allocator — for Limelight, the same
one that serves arenas (`rfc/model/memory/arenas.md`). The substrate does
not implement a page allocator, and it does not use the system allocator
for slabs, because a registered buffer pool must have addresses that are
stable and known.

What the substrate requires of that allocator is small and worth stating
as a contract: a block of a requested size, page-aligned, that stays
mapped until the process ends, and that may be registered with the
kernel. A consumer that cannot promise stability gets an unregistered
buffer pool and loses the fixed-buffer path in the reactor, which is a
performance loss rather than a correctness one.

## Decided elsewhere

| Question | Document |
|---|---|
| what the wait record means and who writes it | `design/execution.md` |
| what a stack costs and when it is released | `design/stacks.md` |
| how buffers are handed out and registered | `design/reactor.md` |
| how an operation is retired early | `design/cancellation.md` |
| what a walk concludes from a smeared read | `design/deadlock.md` |

## Open questions

- **Slab size.** A larger slab means fewer mappings, which matters
  against the ceiling measured in `design/stacks.md`, and a coarser
  growth step. No measurement exists.
- **Whether sockets and timers want separate pools or one.** They differ
  in size and in how often they are walked. One pool is simpler; two
  keeps a walk of timers from touching socket cache lines. Undecided,
  and cheap to change before there is code.
- **Reclaiming slabs.** The design never frees a slab, which is right
  for a server and wrong for a process that peaks once and then idles for
  a day. A slab that has been entirely free across two walks could be
  unmapped, but registered buffers make that unsafe without deregistering
  first, and deregistration is a global operation on some backends.
- **The consumer word.** One opaque word is enough for a pointer to an
  actor context. Whether a consumer needs more, and whether that should
  be a second slot rather than a wider one, waits for a second consumer.
