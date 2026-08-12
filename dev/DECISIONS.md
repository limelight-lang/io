# Decisions

Architectural decisions, newest last. What was decided, why, what was
rejected, and what it costs. Superseding a decision means a new entry,
never an edit of the old one.

## 2026-08-12 — Rust for the core, C ABI at the surface

The substrate is written in Rust, and every foreign consumer reaches it
over a C ABI. Limelight already fixed Rust for its runtime core
(`rfc/runtime/implementation-language.md`), and this repo plugs straight
into that scheduler, so a second language on the boundary would add an
FFI hop to every wake. Completion-based I/O decides it independently:
io_uring keeps the submitted buffer until the completion arrives, and
Rust's type system is what enforces that lifetime instead of author
discipline.

Rejected: C, which would fit php-src but not the Limelight core; C++,
which `implementation-language.md` deliberately confines to the thin
LLVM layer.

Cost: coroutine migration between threads cannot be expressed in safe
Rust and stays an `unsafe` contract of ours (see the scheduler entry).

## 2026-08-12 — Our own scheduler, borrowed primitives

We write the scheduler and take the primitives under it. No external
scheduler mounts an actor: the loop between two tasks must install the
actor's arena into TLS, treat a message boundary as a safepoint, and
deliver the collector's handshake as an ordinary message
(`rfc/runtime/actors.md`). Those are its internals, not its settings.

Taken: `corosensei` for context switching (MIT/Apache-2.0; seven
architectures over ELF, Darwin, Windows and UEFI; public `Stack` trait
for our own allocator), `crossbeam-deque` for work stealing,
`io-uring`/`polling`/`mio` for the reactor backends.

Rejected: Photon and bthread, both Apache-2.0 and both C++, which would
turn the thin C++ layer into a second runtime on the wake path; tokio,
whose task model is stackless and closed to foreign C-ABI callers; `may`,
whose spawn is `unsafe` because a coroutine touching TLS after migrating
threads is undefined — the exact hazard our migrating actors sit on.

Cost: the migration hazard is ours to solve either way. `corosensei`
marks `Coroutine` as `!Send`, so moving one between threads is our
`unsafe` obligation, and TLS may not be cached across a suspension point.

## 2026-08-12 — Two coroutine kinds behind one handle

The scheduler queue holds one handle kind, and the suspension mechanism
under it varies: a stackful context switch, or a state machine driven by
a poll. Languages whose compilers we do not own get the stackful kind,
which needs nothing but a C ABI. Limelight gets the stackless kind where
its compiler can emit the state machine, and pays no stack at all.

Rejected: picking one kind for everything. Stackless alone excludes every
foreign language, because a foreign frame cannot be transformed and
therefore cannot suspend. Stackful alone charges Limelight a stack it
does not need.

Cost: the wake path must not branch on the kind before dispatch, so the
handle carries the discriminant in a spare pointer bit.

## 2026-08-12 — Completion-first I/O API; io_uring is a backend

The API is shaped around completion ("submit this operation with this
buffer, receive the result"), not readiness ("tell me when the fd is
readable"). io_uring and Windows IOCP are both completion-based, and a
readiness backend emulates completion by issuing the syscall when the fd
signals. The reverse direction loses what io_uring is for.

io_uring is therefore the fast path on server Linux rather than the
foundation: Android blocks it from apps through seccomp-bpf, ChromeOS
disables it, and the default seccomp profiles of Docker and containerd
reject it. Backends: io_uring, IOCP, kqueue, epoll.

Cost: the buffer becomes part of the API contract, because the kernel
owns it from submission until completion. Zero-copy send splits this
further — it returns two completions, the result and the
`IORING_CQE_F_NOTIF` that releases the buffer — so "operation finished"
and "buffer returned" are two events in the contract, not one.

## 2026-08-12 — Stacks are reserved whole and committed lazily

A stackful coroutine reserves its stack as one mapping and touches only
the pages it uses. Measured on Linux 6.6 (WSL2, 4 KB pages): 10 000
stacks of 2 MB with one page touched cost 40 272 KB of RSS, or 4.0 KB
per coroutine, against 20 GB of address space. Touching 64 KB of each
costs 64.0 KB per coroutine. The reserved size is address space, and
64-bit address space is not the scarce resource.

Rejected: segmented or chunked growth. It needs a prologue check in
every function, which a foreign ABI does not have, and the guard-page
fault arrives too late to switch stacks — `sub rsp, N` has already run
and the faulting instruction addresses the frame it just made. The only
resumable answer is to map a page where the access happened, which is
in-place growth, which is what reserving already gives.

Cost: mappings, not memory. With one guard page per stack the ceiling is
`vm.max_map_count / 2` — measured as 32 754 stacks against the default
65530. Past that: raise the sysctl, or carve stacks out of slabs and
detect overflow on return to the scheduler instead of by guard page.

Open: a frame larger than a page can step over the guard (stack clash).
Either probe large frames, as `gcc -fstack-clash-protection` does, or
make the guard band as wide as the largest frame we emit.

## 2026-08-12 — Deadlock detection: three constraints fixed, the mechanism open

The mechanism is undecided and is the subject of `design/deadlock.md`
(PLAN.md, S4.1). Candidates: an explicit wait-for graph maintained at
park time, or a scan over the object pools that reconstructs the same
relation on demand. Three constraints hold under either.

**Quiescence is not the criterion.** `php-src/ext/async` uses it:
`resolve_deadlocks()` fires only when the whole process has no active
events and no runnable coroutines. A cycle of three coroutines among two
hundred busy ones never surfaces, and quiescence does not prove a cycle
either, which is why that code carries a last-chance reactor drain
against false positives. Detection here starts from wait age instead, the
way PostgreSQL arms `deadlock_timeout` (1 s by default) rather than
checking on every wait.

**Both halves of a wait are recorded.** `ext/async` stores what a
coroutine waits for (`waker->events`) but not who will post it, and
without the second half no cycle can be closed — which is why its
channels fall back to 5 s timeouts and a bulk close. Every suspension
goes through a single park primitive that records both. A wait registered
anywhere else is a hole, and the hole is not self-diagnosing.

**Detection may not depend on the collector finishing a cycle.** A
deadlocked actor never reaches a message boundary, so it never answers
the collector's handshake: a deadlock stalls mark termination, and a
detector waiting on the collector would be unavailable exactly when it is
needed. It needs no handshake of its own, because it reads only parked
units and a parked unit does not mutate. Sharing the collector's walker,
header discipline and epoch bytes stays on the table; sharing its trigger
does not.

Kept from `ext/async`: binding a channel to the scope that created it.
That one is driven by lifetime rather than by a timeout.

## 2026-08-12 — One repository, three crates

The reactor is separated from the execution substrate as a crate, not as
a repository. Separation earns its keep: the backends are platform code,
and a consumer running computation without I/O should not compile them.
Keeping both in one workspace keeps the wake contract — a completion
resumes a unit — changeable in one commit, and that contract is
co-designed with the unit rather than layered on top of it.

Layout: `io-core` (unit, switch, stack, sched, pool, deadlock),
`io-reactor` (submission, completion, four backends behind cargo
features), `io-api` (public surface and the C ABI). Dependencies point
from `io-api` down to `io-reactor` down to `io-core`; the reverse edge
does not exist.

Rejected: `io-rt` as the first crate's name, because it abbreviates
"runtime" and that word already names two other things in this project;
`io-api` for the same crate, because it names a shape rather than a
subject, and because the facade is what actually deserves it.

Cost: the wake contract becomes a published API between crates, so
changing it means a version bump even inside one repository. That is the
price of letting a consumer skip the backends, and it falls on the
boundary we most want stable anyway.

## 2026-08-12 — Actors run on stackful units; stackless is the special case

An actor always runs on a stackful unit. Stackless units stay available
for cases that never need what a stack buys: a body the compiler
transforms whole, which neither hosts an actor nor suspends below a
foreign frame.

The split is therefore not symmetric. Stackful is the default and carries
the actor contract, because an actor may suspend at any depth — a
synchronous call to another actor parks mid-message, and the frames below
that point are ordinary compiled frames, some of them foreign. A
stackless unit cannot suspend there at all: there is no way to return
"pending" through a C frame.

What this buys: the configuration that would otherwise need a policy —
a stackless unit re-entered through foreign code that then awaits —
becomes an error instead of a question. Limelight's compiler rejects it
statically; the C ABI rejects it at runtime.

Cost: two mount paths in the design, not one. Only the stackful path
installs an actor context, so the stackless path must be readable as a
restriction of it rather than as a second mechanism.

## 2026-08-12 — Nothing the kernel touches lives on a unit stack

Buffers, `iovec` arrays, `msghdr` structures — everything the kernel may
read or write after an operation is submitted — come from the buffer pool
and never from a unit's stack. The submitting API is where this is
enforced, because it is the only place that can.

Completion-based I/O separates "the call returned" from "the kernel is
done", and a stack-resident buffer stays exposed for the whole gap.
Cancelling the unit does not cancel the kernel's claim, and the retired
loser of an `await` with a timeout is the common case rather than the
exotic one. Three corruptions follow from allowing it, and none is
detectable: a late completion writing into a pooled stack now owned by
another unit; a late completion writing into the *same* unit's live
frames after it returned past the awaiting frame; and either of those
arriving after the process has forgotten the operation existed.

What the rule buys is that a stack's lifetime is exactly its unit's.
Without it, stack management needs a per-stack in-flight count that
survives unit teardown, an owner for a stack that belongs to no unit and
no free list, and a policy for a completion that never arrives.

Cost: a consumer cannot read into a local variable. The buffer contracts
(`design/reactor.md`) are the API that replaces it, and one of the three
already hands the buffer out from a pool rather than taking one.

## 2026-08-12 — The wait-for relation is the wait records, traversed on demand

Deadlock detection follows `poster` links from one parked unit's wait
record to the next. There is no separate graph structure and no scan of
the population: the records already hold both halves of every edge, so a
maintained graph would be a copy that can disagree with the truth, and a
scan would cost the number of units rather than the size of the question.
This settles the mechanism left open on 2026-08-12.

The two wait modes make one algorithm cover both cases the literature
separates. Assume every parked unit reachable from the suspect is
blocked, then remove any unit that is not — an OR unit with any live
half, an AND unit with every half live — and repeat to a fixed point.
Under AND alone that finds a cycle; under OR alone it finds a knot; the
modes are in the records, so nothing has to choose between the two
algorithms.

Rejected: quiescence as the criterion, which is what `php-src/ext/async`
uses and which cannot see a cycle inside a busy process.

Cost: a full pool scan still exists, but only for the human-facing
diagnostic dump, and it is not on the detector's path.

## 2026-08-12 — Correction: a zero-copy send does not always post two completions

The completion-first entry above states that `IORING_OP_SEND_ZC` returns
two completions. That is true only when the first completion carries
`IORING_CQE_F_MORE`. A send that fails posts one completion and no
notification ever follows.

The decision it was given as a reason for stands: "operation finished" and
"buffer returned" are still two events in the contract. What changes is
how many the code waits for — an operation's owed-completion count is read
from the first completion's flags rather than fixed at two
(`design/reactor.md`, `design/pool.md`). Fixing it at two strands one pool
buffer on every failed zero-copy send, and under connection churn the pool
drains and all I/O stops.

Recorded as its own entry rather than as an edit, because the original
reasoning is what a reader will meet first and the correction has to be
visible next to it.

## 2026-08-12 — A coroutine is an entity, always in the GC heap

A coroutine is an ordinary runtime entity: an `RcHeader` at offset zero,
its own entity kind, allocated by the Limelight memory manager
(`limelight-lang/model`). Its lifetime is its reference count, the
existing heap enumerator lists it, and the existing tracer walks its
children. The stack and registers are a **separate** pooled object, as
`async_fiber_context_t` is in TrueAsync: stacks are expensive and
reusable, coroutines are cheap and numerous.

Category: the GC heap, always, even though a coroutine could live in a
request arena like any other object. Choosing one category removes the
question of what happens when a reference to a coroutine is held from
outside while its arena dies — and a parked coroutine is referenced from
outside by construction, because something has to wake it.

Cost: one heap allocation per message rather than a bump in the arena,
and a coroutine is subject to cycle collection instead of dying with its
arena in O(1). Both are accepted for the first implementation and are
revisitable once there is something to measure.

Rejected, and this is a correction rather than a choice: a bespoke array
of fixed slots addressed by index and generation, which earlier documents
in this repository describe. It solved "how does a late wake know its
coroutine is gone", and the reference count solves that already — whoever
armed a half holds a reference, so the coroutine cannot die underneath it.
Deferred reclamation went with it. What survives is the parking protocol
itself: four states, the order state-record-arm, the seqlock over the wait
record, and the epoch identifying one wait. Those are about races inside
one coroutine's life, not about its lifetime.

## 2026-08-12 — A coroutine lives where its memory lives, which splits it in two

Supersedes the entry above that put every coroutine in the GC heap. The
category follows the memory the coroutine works with, and there are two
answers because there are two kinds of memory.

**An ordinary coroutine shares its thread's memory**, so it is allocated
in that thread's heap and never leaves the thread. This is not a
scheduling policy: its reference counts are non-atomic, and resuming it
on another thread would have that thread reach into a heap that is not
its own. It is created on whichever thread creates it, and it dies there.

**An actor shares memory with nobody**, so an actor's coroutine is
allocated in the actor's arena and travels with it. The actor may be
mounted on a different thread for a later message, and its arena moves
with it, so the coroutine object never becomes memory of a thread that no
longer runs it.

Consequences worth naming, because they constrain everything above:

- **Load cannot be balanced by moving ordinary coroutines.** They are
  pinned by construction, so a thread that receives a burst of long
  coroutines keeps them. The unit of load movement is the actor, which is
  what the concurrency model already says it is.
- **Work stealing applies to actors only.**
- **The single `unsafe impl Send` shrinks to the actor path.** An ordinary
  coroutine never crosses a thread boundary, so the rule against caching
  a thread-local address across a suspension point, and the pinning that
  a live foreign frame imposes, are obligations of the actor path alone.
- **A completion can arrive on a thread that does not own the coroutine**,
  because each worker has its own ring. It is forwarded to the owner's
  intake queue, which is the same path a cross-thread cancel already
  takes (`design/reactor.md`).

## 2026-08-12 — The coroutine object, settled

Agreed with Edmond in conversation, and this entry is the summary the
architecture map (`dev/ARCHITECTURE.md`) will be written against.

- A coroutine is an **ordinary object**: entity kind 0, with a class the
  runtime builds. No new entity kind is spent — code 7 is the last one
  free and stays free. Tracing, teardown and PHP visibility all come from
  the object machinery unchanged. TrueAsync does the same:
  `async_coroutine_t` carries a `zend_object` and has a class entry.
- **The waker is embedded in the coroutine**, as it is in TrueAsync, for
  the reason its comment gives: no allocation on the parking path. Two
  wait halves sit inline, which covers the common one-or-two-resource
  case; the half count is the discriminant, so `count > 2` means the same
  field holds a pointer to one raw block sized to the count. No separate
  flag, because a flag can disagree with the count.
- **The spill block is raw memory, not an entity.** It has exactly one
  owner and a strictly nested lifetime, so a reference count would buy
  nothing and cost two atomics per park. It is freed through
  `deferred_free`, because the collector records the addresses of the
  cells it holds and re-reads them in a later phase.
- **Its cells reach the tracer through a `walk` hook** — a new optional
  field beside `dispose` on the class descriptor, requested of
  `limelight-lang/model` as its stage S17. The hook yields cells through
  the reader rather than children, because the collector re-reads a
  cell's address and raw word; it is called from
  `object::for_each_counted_cell` after the runs, so tracing, cycle
  collection and teardown all inherit it from one place; and it is copied
  into a subclass descriptor the way `dispose` is, or a PHP subclass of
  `Async\Coroutine` silently loses waker tracing.
- **The stack and registers are a separate pooled object**, not an
  entity, exactly as `async_fiber_context_t` is in TrueAsync: stacks are
  expensive and reusable, coroutines are cheap and numerous.
- **The Scheduler owns a coroutine's lifetime** and holds a reference to
  it, which is also what keeps a cycle of blocked coroutines from being
  collected as garbage before the deadlock detector sees it.

What this retires from the earlier documents in this repository: the
64-bit handle carrying a slot index and a generation, the bespoke slot
array, and the deferred reclamation that made a late wake safe. A
reference count does all three jobs — whoever arms a half holds a
reference, so the coroutine cannot die underneath it. What survives is
the parking protocol itself: the four states, the order state-record-arm,
the seqlock over the wait record, and the epoch identifying one wait.

## 2026-08-12 — Re-mounting on another thread: the shape is fixed, the rule is open

**Only an actor ever moves between threads.** An ordinary coroutine
shares its thread's memory and is pinned for life, which is a
consequence rather than a policy (entry above).

**An actor moves only where it has stopped** — at the points where it
reads from its mailbox, and where it is waiting. Not at an arbitrary
instruction, and not merely because a completion happened to arrive on
another thread's ring.

Open, and to be researched rather than decided now:

- exactly which points qualify. "Reading from the queue" is the message
  boundary the actor model already names as a safepoint; "waiting" is
  broader and needs to be enumerated, because a wait mid-message leaves
  live frames and whatever the message has touched so far.
- whether a per-suspension-point flag is needed at all. The argument that
  it is not: for code we compile, nothing thread-affine survives a
  suspension point by construction — no thread-local address is cached
  across one, and a suspendable path carries no callee-saved registers
  (`design/switching.md`) — so the only veto is a live foreign frame, and
  the foreign-frame counter already answers that. The argument has not
  been checked against real generated code, which is what makes this a
  research item.
- what a consumer over the C ABI declares, and when. Its code is opaque
  to us, so the safe default is that it does not move, matching the
  pinning default in `design/execution.md`. Whether the declaration
  belongs on the actor at creation or somewhere finer is undecided.

Until this closes, the implementation forwards every completion to the
owning thread. That is correct under either answer and costs the
forwarding path, which exists anyway for cross-thread cancels
(`design/reactor.md`).

## 2026-08-12 — A coroutine is single-threaded; the detector rides the collector

Two decisions that belong together, because the first is what makes the
second necessary.

**Only its own thread ever touches a coroutine.** A signal from another
thread is delivered to that thread's Reactor, which runs on that thread
and touches the coroutine there. The coroutine is never the cross-thread
door.

What this retires from the design documents, which still describe it: the
compare-and-swap on every parking transition and the `Parking` state
itself, the seqlock over the wait record, and the generation check on the
wake path. All three existed to resolve a race between a waker on another
thread and a coroutine still suspending, and that race does not exist.
Parking becomes ordinary stores. The **epoch survives**, for its other
job: telling this wait from the previous one, so a retired half that
fires anyway does not wake the coroutine out of an unrelated wait.

**Deadlock detection rides the collector.** One pass does both: it finds
cycles and it frees memory. The collector already reads other threads'
entities safely — block snapshots, epoch bytes, a re-read of each
recorded cell in a later phase — and the detector inherits that instead
of duplicating it. The single-thread invariant stays intact, because the
collector is the one reader that crosses by right.

**Detection runs in the walk phase, never after mark termination.**
Termination waits for every actor to answer a handshake, and an actor
parked in a deadlock never reaches a message boundary, so it never
answers. A detector placed after termination would be blocked by exactly
the condition it exists to find. The walk needs no answers: it reads
snapshots.

Open:

- **The trigger.** A deadlocked set allocates nothing, so nothing about
  it makes the collector run. Either the wait-age watchdog also starts an
  epoch, or the collector runs on a schedule of its own. Undecided.
- **Latency.** Detection now happens at collector granularity rather than
  at wait-age granularity, which may be seconds rather than the second
  PostgreSQL uses. Whether that is acceptable is a question for Edmond
  and not for the design.
- **What the detector needs that the collector does not.** The collector
  reads counted edges; the detector reads the wait record — the resource,
  the mode, which halves have fired. Whether the collector's walk yields
  those for free through the `walk` hook, or the detector needs a second
  reading pass, is not worked out.
