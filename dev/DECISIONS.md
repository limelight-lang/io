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
- **Whose thread the collector runs on**, which decides what the detector
  can see. On its own thread, watching the mutator threads, it reads
  every thread's entities and a cross-thread cycle is visible in one
  pass. On a mutator's thread it sees that thread's memory only, and a
  cycle spanning threads has to be assembled from several passes, which
  brings back the combining step and its stall on a thread that does not
  answer. `ll-model` has a threaded driver today (`run_epoch`) and a
  steppable collector beside it, so both remain open. Raised by Edmond
  2026-08-12 and explicitly left to think about.

## 2026-08-12 — The collector runs on its own thread

Decided by Edmond, closing the last open item of the entry above. The
collector has a thread of its own and watches the mutators, so it reads
every thread's entities in one pass and a cycle spanning threads is
visible without assembling partial answers. The rejected alternative —
running on a mutator's thread — sees that thread's memory only, which
brings back the combining step and its stall on a thread that does not
answer, and the detector cannot afford that stall because the thread that
does not answer is the one it is looking for.

## 2026-08-12 — `half` is renamed `entry`

A wait record holds one **entry** per thing a coroutine waits on: the
resource handle, a cancel handle, a result slot, a fired bit. The kind of
the resource is read from the resource, never copied into the entry.
The earlier name was `half`, from "half an edge", and it was wrong in two
ways: an edge has exactly two halves while a record has as many entries as
the coroutine waits on things, and the record's own field table already
called them entries. `wake(unit, half, epoch, result)` becomes
`wake(unit, entry, epoch, result)`.

The word survives where it is true. A wait edge has two ends: the
coroutine's end names a resource, and the resource's end answers who can
end the wait — a debtor by naming it, a channel or a future by who can
still reach its write end. Neither end is a record field.

## 2026-08-12 — The detector: a liveness fixpoint over one collector walk

Supersedes the trigger, the search and the victim rules of
`design/deadlock.md`, which described a per-worker search from a suspect
under a seqlock. The mechanism below is what that document now carries;
this entry records what was chosen and why.

**Resources split three ways, and the kind decides the rule.**

- *External* — a kernel operation, an armed timer. Served without any
  coroutine, so such an entry is always live.
- *Debtor* — a mutex, a synchronous actor call, a join on a coroutine, and
  a semaphore whose permits are released only by dropping a guard. The
  resource names who owes it, read fresh at evaluation.
- *Supply* — a channel, a future, an actor mailbox, and a semaphore anyone
  may post without acquiring first. No owner field can exist, because
  whoever holds the write end may serve the wait. Liveness is reachability
  of the write end.

The semaphore is split by its API rather than by its name, and the split is
not cosmetic: a semaphore created with zero permits and posted by a
producer that never acquired anything has no holders, so calling it a
debtor would report its waiter deadlocked while the producer runs. Release
tied to acquisition is what makes a holder a debtor, and only a guard
enforces that tie. Everything reached over the C ABI counts as postable,
because its discipline is not ours to see.

The third kind is what the earlier document missed, and Edmond raised it:
a future is a memory cell that someone may write. A coroutine awaiting one
that nobody will resolve waits forever while belonging to no cycle, so a
cycle search cannot see it. Reachability can, and the detector reads
reachability anyway because it runs inside the collector's walk.

**Detection is a least fixpoint over the whole heap, not a search from a
suspect.** Seed the live set with what proceeds unaided: every coroutine
that is not parked, every parked coroutine whose wait is already decided,
every entry the kernel or the timer wheel will end. Then grow: a debtor
entry is live when its debtor is live, a supply entry is live when the
resource can serve it now or its write end is reachable from something
already live; a coroutine in OR mode is live when any outstanding entry is
live, in AND mode when all are. Iterate until the live set stops growing.
Whatever remains parked and unmarked can never proceed.

Growing from the live set is the direction that works. Starting from "all
live" and removing lets a cycle count itself as its own support, so it is
never removed.

**Liveness is per entry, never per coroutine.** An earlier form of this
rule marked a whole coroutine live because one of its entries waited on
the kernel, and that loses a real deadlock permanently: B waits on AND
[kernel operation, mutex N held by A] while A waits on mutex M held by B.
The kernel answers and B still never proceeds, because N arrives only from
A and A waits on B. Under the per-coroutine rule B is live, liveness
reaches A through N, and the pair is never reported.

The same correction fixes the arming rule of `design/deadlock.md`, which
armed no watchdog for a wait with any kernel entry. Call an entry
cycle-capable when it names a mutex, semaphore, channel, actor, future or
join. An OR
wait arms a watchdog when every entry is cycle-capable, since one kernel or
timer entry satisfies an OR forever; an AND wait arms one when at least one
entry is cycle-capable, whatever else it names. Both parts of that rule
are load-bearing: without the second, a cycle through a mutex held beside a
kernel wait arms nothing; without the first, three coroutines in OR over
each other's mutexes form a knot that arms nothing. The cost promise
survives: a socket read is a single kernel wait or an OR wait with a
timeout, so a hundred thousand idle connections still arm nothing.

**Two marks are propagated by one walk.** The memory mark keeps its roots,
including the scheduler's ownership table, the timer wheel and the reactor
intake queues. The liveness mark takes different roots — globals, the C-ABI
handle table, unparked coroutines, decided waits, coroutines named by a
pending intake entry — and does not flow out of a parked coroutine that is
not itself marked live. Rooting liveness in the scheduler's table would
mark every parked coroutine reachable and blind the detector permanently.

Beside the two marks the pass collects a **served set**: a resource that a
pending intake entry would move so a wait on it can proceed, a deposit and
a take alike. It is a set and not a mark, because it says that this
resource can serve a wait now, which is not what reachability of a resource
would say. Without it, a cross-thread send whose sender has already
finished is recorded nowhere the pass reads, and its receiver is reported
dead. It obliges the worker to hold one intake queue
drained in order, so a deposit recorded during a pass is applied before the
resolution that pass produced; `design/reactor.md` owes that contract.

**The trigger is the wait-age watchdog, with quiescence as an
accelerator.** A watchdog expiry requests a collector epoch rather than
running a search itself. The last worker about to sleep requests one too,
which detects a total freeze at the moment it forms instead of a threshold
later; that is what Go's `checkdead` gets right, and its limitation is
that it has nothing else, so a cycle among three goroutines inside a busy
program is never found. Quiescence stays an accelerator here and never a
criterion, which the entry of 2026-08-12 above already required. Threshold:
1 s in PostgreSQL's spirit, doubled on each re-arm of the same wait epoch
so a long healthy mutex hold costs a thinning series of passes. The figure
is not measured and stays a deployment parameter.

**Deadlock resolves softly: the waiter resumes with an exception.**
Requested by Edmond. A proved-unresolvable wait fails at its wait point
and the process keeps running. A channel receive fails exactly as a
receive on a closed channel fails, and a future await exactly as a broken
promise fails, so code already correct against closure needs no change;
the error value carries the cause and the report handle. Mutex, actor call
and join have no closure to imitate and raise a deadlock error of their
own.

**The detector marks no resource.** A resource dies by its own rule:
dropping the last write end closes a channel and breaks a future, and that
wakes every waiter through the ordinary path. The consequence Edmond asked
for arrives anyway, because the resumed coroutine drops the last write end
while unwinding, which closes the channel and fails its co-waiters at once
without a second collector pass. A mark placed by the detector would be
wrong in the case that matters: while a write end is still held by
coroutines of the dead set, resolving one of them may resurrect a
legitimate writer, and the mark would turn its legitimate write into an
error the detector itself invented.

**One coroutine per sink strongly-connected component is resolved per
pass.** Resolving one invalidates every deadness proof that depended on it,
and a sink's proofs depend only on its own members and on facts grounded
outside the set, so a sink is the largest batch that stays sound. A cycle
is a sink of two or more; a coroutine starved by an unreachable write end
is a sink of one. Bystanders hang above the sinks and are never chosen,
which the earlier document already required.

**The collector never writes into a coroutine.** It posts a conditional
resolution to the owner's reactor, and the owner acts only if state, wait
epoch, winner and fired bits still equal what was recorded. Every input to
deadness except the fired bits and the winner is written once per epoch,
and those two only grow within one, so equality proves the snapshot
described this exact wait. This is what makes a false kill require a
broken contract rather than a lost race, and it is why parking can stay
plain stores: a second writer would recreate the cross-thread race the
single-thread invariant removed.

**Policy is the consumer's, process-wide:** soft by default; hard, which
publishes the report and aborts, for tests and CI where a deadlock must be
loud; report-only, which publishes and resolves nothing, for observation
in production.

**The design errs toward missed detection, and soft resolution does not
move that.** A false positive is a semantic lie rather than a survivable
hiccup: one receiver observes a closed channel while another reads data
from the same healthy channel, and no catch block repairs that. The
validation costs nothing measurable per pass, so relaxing it would buy no
speed either, and latency is set by the threshold instead.

What this retires: the seqlock over the wait record, the claim word and
the two-search race it settled, the per-worker search from a suspect, and
the rule that any kernel entry disarms the watchdog. What it adds as an
obligation elsewhere: closing on the drop of the last write end is channel
and future semantics, and no document owns it yet.

Open:

- **Integration with the `ll-model` walk is asserted, not verified.** The
  detector needs a second mark with different flow through parked
  coroutines, a re-visit of a coroutine that turns live mid-fixpoint, and
  multi-word snapshots read across threads — of a wait record, and of a
  resource's own fields such as a mutex holder or a channel's occupancy and
  closed flag — under a re-read discipline built for single counted cells.
  Whether `run_epoch` and the steppable collector host all three is
  unchecked against their code. The fallback, if they
  cannot, is a detector-owned second traversal of parked coroutines inside
  the same pass, which changes the cost and not the protocol.
- **A live holder that never writes** is undetectable by construction:
  reachability proves possibility, not intent. The diagnostic dump over
  the pools is the tool for that case.

## 2026-08-13 — Correction: `ext/async`'s channel timeouts are opt-in now

The entry of 2026-08-12 on deadlock detection says that `php-src/ext/async`
falls back to five-second timeouts and a bulk close, and gives that as a
reason. The reason stands; the tense does not. Verified in the working copy at
`~/php-src/ext/async`: `5000` appears in no `.c` or `.h` file, the generated
`channel_arginfo.h` shows the constructor's defaults as `0`, `0`, `false`, and
`docs/channel-deadlock-protection.md` still documents 5000 ms as shipped
behaviour. The defaults were reverted to zero, which disables both timeout
layers, so out of the box a channel deadlock there surfaces as a generic
`DeadlockError` with a wait-graph dump.

The revert's own message is a stronger argument for this substrate than the
timeout ever was: the 5000 ms defaults "fire false positives in normal pub-sub
patterns where a producer is a coroutine that calls send() occasionally rather
than blocking in it", so after five idle seconds the channel closed itself
while its producers were alive and might have produced later. A wall clock
cannot tell "no producer exists" from "the producer is idle", and that is a
category error rather than a tuning problem — which is exactly what
reachability of the write end answers instead.

What survives from that codebase is the third layer, which never had to be
reverted: binding a channel to the scope that created it, as observation
rather than ownership, with a back-pointer so either side may die first.

Recorded as its own entry rather than as an edit, for the reason the
zero-copy correction of 2026-08-12 gives.

## 2026-08-13 — An actor may be re-mounted anywhere; the difference is price

Decided by Edmond, and it replaces the conservative rule in
`design/execution.md` that re-mounted an actor only at a message boundary.
**Every suspension point is a re-mount point. What differs between them is
what the move has to carry.**

**Between messages the move carries nothing.** The actor has no unit, no
stack and no saved registers: one message is one unit, created when the
message is taken from the mailbox and destroyed when it is done. Mounting
elsewhere installs the arena into that worker's thread-local storage and
takes the next message. Beyond a cold arena there is no cost at all, which
makes this the default balancing point.

**Inside a message the move carries the context.** The saved live-register
set and the live stack travel with the unit, and both are memory, so the
mechanism is the same handoff with the same release and acquire pair. Two
costs are real. The cache is one: the stack's hot pages and the arena are
cold on the new core, and the deeper the frames the more of them are warm.
The other is that everything already armed points at the previous worker.
An operation submitted to that worker's ring completes there, a timer sits
in its wheel, a cross-thread cancel is addressed to its intake queue, and
the detector addresses a resolution to the owner it recorded. Each becomes
a forwarding obligation, and `wake` running "on the unit's own thread"
becomes a claim on a mutable owner field rather than a fact, because
parking is plain stores precisely on the strength of there being one such
thread. That is what a mid-message move buys with.

**What it buys is worth the price for our profile.** An actor that parks a
hundred times on I/O within one message would otherwise be tied to its
first worker for all hundred resumptions, and it is the resumption, not the
park, that needs a free core.

**The price is paid only where it was asked for.** Decided by Edmond after a
Critic round costed it: an actor declares whether it may move mid-message,
and one that does not moves only between messages and keeps plain stores on
its own thread. A declaring actor moves at most once per message. The bound
is what keeps the cost proportional to the benefit, because the profile this
exists for parks a hundred times inside one message, and a move at every park
would pay the claim and the shared structure's mutex a hundred times over
(see the scheduling entry of the same date).

**One veto is absolute and is not a price.** A unit whose foreign-frame
counter is non-zero does not move at any point. The frames below hold state
tied to the thread — `errno`, an OpenSSL error queue, a locale handle, a
foreign allocator's caches — and an OS mutex taken there is owned by the
thread that took it, so releasing it from another is undefined
(`design/switching.md`). A unit that opted out of maintaining the counter
reads as permanently non-zero and therefore never moves.

**What the counter does not cover has to be said in the same breath.** It
sees nested foreign frames and nothing else. Thread-affine state that a
library takes in one call and releases in a later one is invisible to it:
memory handed out from a per-thread allocator cache on one worker and freed on
another, a `flockfile` and `funlockfile` pair split across two calls, any
handle a library parks in its own thread-local storage and reuses on the next
entry. Between those calls the counter reads zero and the unit is free to
move. `design/execution.md` holds this open under pinning granularity; the
honest statement is that the veto is exact for what it measures and silent
about what it cannot see, and that a consumer who knows it keeps such state
declines the mid-message declaration — the same lever as opting out of the
counter.

**The claim is the state word, widened — ruled by Sage, and it is not a lock
a forward holds.** A lock would stall mounting on another thread's progress.
For a declaring actor the unit's state word becomes one atomic word carrying
the state and the owner: `Running(W)`, `Parked(W)`, `WokenLocal(W)`,
`WokenShared`, `Terminal`, with the cancelled bit moved to an atomic byte
beside it so that mounting needs no compare-exchange loop to preserve it. For
every other unit the word stays what it is, plain stores with a fixed owner.

**Delivery is one rule, applied by whoever holds a signal** — a completion on
a foreign ring, a cancel, the detector's resolution — and applied again when
a worker drains its intake queue. Read the word once, then:

- it names me: apply locally;
- it names another worker: put the payload in that worker's intake queue and
  ring its wake descriptor;
- `WokenShared`: drop a wake and drop a resolution, and write the byte for a
  cancel. Dropping is correct because `Woken` means the wait is already
  decided: a wake on a decided wait is a no-op, and step 12 of
  `design/deadlock.md` requires the state to still be `Parked`. Nobody writes
  the record in that window;
- `Terminal`: drop, and release whatever the signal held.

Only the wake's payload travels — the unit, the entry index, the epoch, the
result. An operation's accounting stays in its slot on the ring that
submitted it and never moves.

**Against the protocols this changes two steps and adds one.** Dispatch is
step 0 of `wake`, ahead of the epoch check, so a foreign thread never reads
the epoch. Steps 1 to 4 are unchanged and stay plain stores, because the
thread applying them owns the unit. Step 5 splits: locally, store
`WokenLocal(W)` and enqueue into the worker's own list; on the move path,
where the actor declared it, the foreign-frame counter is zero and the limit
is unspent, store-release `WokenShared`, then push to the ready set, then ring
a sleeper. Word before queue, and after that store the worker touches the unit
no more — including its own later intake entries about it, which dispatch will
catch. Mounting is the acquisition: take from the ready set, compare-exchange
`WokenShared → Running(W')` with acquire semantics, install the arena,
continue. Parking is untouched except that step 4's store is atomic and the
word carries the owner; arming always targets the current worker's ring and
wheel, and the one-move limit resets when a unit is created.

**What weakens, exactly.** "Only the unit's own thread writes the record"
becomes "only the thread the claim word names writes the record, and handing
ownership over carries release and acquire" — the same pair that already
publishes the arena. Inside an ownership window there is one writer; inside
`WokenShared` there are none. The detector is unaffected: a move requires a
wake, a wake claims the winner and sets a fired bit, so step 8 or step 12
discards a finding that predates it, and the next park writes a fresh epoch.
The served set holds, because forwarding only delays a resolution against a
deposit and never advances it.

**Cost.** For a declaring actor: an atomic store per transition of the word,
one compare-exchange when a moved unit is mounted, an atomic byte for cancel.
For every unit, declared or not: one atomic load and a branch each time an
intake entry is applied. For a signal that lost the race against a move: one
scheduler turn of extra latency. For everyone else: nothing, and
`design/execution.md`'s statement about the unit's own thread stays a fact for
them and becomes conditional only for the declared.

**A declaration and the counter's opt-out are incompatible**, and the pair is
rejected when the actor is created rather than discovered at runtime: opting
out reads as a permanently non-zero counter (`design/switching.md`), so such
an actor would declare a price it can never pay.

## 2026-08-13 — The wait epoch is written once, at park

Three documents gave two answers, and `dev/INDEX.md` named the
disagreement as an input to the detector's correctness rather than a matter
of wording. `design/cancellation.md` moved the epoch when a wait ended,
"which is why the bump belongs to the winner rather than to the next
parking"; `design/pool.md` had it change "when it *parks again*, not when
it wakes"; the wake protocol in `design/execution.md` has no bump step at
all. The answer is the last two: **the epoch is written in step 1 of the
parking protocol, together with the rest of the record, and nothing on the
wake, cancel or resolution path touches it.**

Two facts decide whether a signal is accepted, and they answer different
questions. The epoch says which wait the signal belongs to. The winner
field under OR, and `remaining` under AND, say whether that wait is still
undecided. A late signal from a previous wait is rejected by the epoch,
because the unit has re-parked and re-written it. A late signal from the
current wait, already decided, is rejected by the winner or by the
counter — and that test has to exist anyway, since two entries of one AND
wait fire legitimately and two entries of one OR wait race legitimately.
Bumping the epoch at the end of a wait would add a second mechanism for
what the decided test already covers, and it would give the record's
identity three writers instead of one: the winner of an OR wait, the cancel
path, and the detector's owner-side resolution each end a wait.

What this corrects in `design/execution.md`: step 4 of the wake protocol
says retirement is safe because of "the epoch check in step 1", and within
one wait it is not the epoch that rejects a retired entry's late fire but
the decided test in step 3. The epoch covers the case its own step 1
describes, where "the unit has since parked on something else". Both
sentences are true of different windows and the document credits one
mechanism with both.

Found while settling this, and it belongs to the same correction: step 3
must test whether the wait is already decided instead of inferring it from
the arithmetic. A retired entry that fires after `remaining` reached zero
decrements it to a negative value, which no reader is prepared for.

Applying all of it to `design/cancellation.md`, `design/execution.md` and
`design/pool.md` is step S5.3. `design/pool.md`'s sentence goes with the
whole of its unit-walking section, which `dev/INDEX.md` already retires:
the detector walks the collector's object graph, and a unit is no longer a
pool slot.

## 2026-08-13 — Scheduling an actor: one ready set, private lists, no thief

Supersedes `crossbeam-deque`, taken on 2026-08-12 without a comparison,
and supersedes this entry's own first form, which put ready actors into
per-worker queues and then needed shedding, lanes and a vetting rule to get
them back out. That machinery answered a question that does not exist.
**What is distributed is the right to run one message.** A ready actor in a
queue is a pointer in a list, not a resident of a thread, so nothing has to
be moved between queues; a worker only has to decide which actor's next
message it mounts.

**Two structures exist because two kinds of memory exist, and this is not a
locality choice.** An ordinary coroutine has non-atomic reference counts in
its thread's heap and is pinned for life, so its queue is private to its
worker and needs no atomics at all. An actor carries its arena and may be
mounted anywhere, so only actors reach a shared structure. Vyukov's 2014
NUMA-aware scheduler proposed the same split for Go — "work I would like to
do myself", single-producer and lock-free, beside "work I am ready to
share" — and it was never implemented, because affinity is a whole-runtime
property and the runtime had shipped without it. Here the split follows from
the memory model instead of being retrofitted onto it.

**What is queued, and what is not.** One entry per ready actor, and never a
message. The mailbox belongs to the actor, is never shared, and its depth is
invisible to the queue: three messages and three hundred thousand put the
same single entry into it. A mounted actor is in no queue at all, which is
why queue length alone is never the load signal — BEAM stalls exactly there,
two full-time processes on one scheduler holding the run-queue length at 1
forever while a second scheduler stays out of work (erlang/otp#9762, the
revert of "don't steal a lone task").

**The actor's readiness word is the arbiter.** It is named apart from the
unit's state word deliberately: `Running`, `Parked` and `Woken` belong to a
unit and are plain stores on its own thread, while an actor's readiness is
atomic, because any thread may post to its mailbox. Three values and five
transitions, and the table has to be complete, since a missing transition is
how an actor comes to run on two workers at once.

| Who | Transition | When |
|---|---|---|
| a sender, on any thread | empty → ready | after pushing into the mailbox; the winner enqueues the actor into the ready set and the loser does nothing |
| a worker | ready → running | before it takes a message; whoever loses this does not mount |
| a worker | running → running | it keeps the actor and takes the next message, while its message budget lasts |
| a worker | running → ready | the budget is spent and the mailbox is not empty: it enqueues the actor and lets go |
| a worker | running → empty | the message is done and the mailbox reads empty |

**Only the winner of a transition into running may mount, and a worker that
has just published empty is no longer a winner.** It re-reads the mailbox
after publishing, because a message posted inside that window leaves an
actor nobody enqueued, and that hang is unreportable: the detector reads a
non-empty mailbox as proof that the wait can still be served
(`design/deadlock.md`). What the re-read produces is an enqueue and never a
mount — the worker attempts empty → ready itself, exactly as a sender would,
and stops if it loses. Mounting on the strength of a re-read is the
two-worker bug, because a sender may have won empty → ready in the same
window and another worker may already hold the actor.

**The readiness word stays `running` for the whole message**, across every
park inside it, which is what keeps a sender from putting a second copy of the
actor into the ready set while a unit of it is in flight. A ready-set entry
therefore means one of two things, told apart by the unit's own word: take the
next message from the mailbox, or mount the unit that reads `WokenShared`.

The window closes only under the orderings that go with it: the mailbox push
is a release store, the re-read an acquire load, and the readiness word is
sequentially consistent. With a relaxed store on either side the message is
lost for good, in the way the paragraph above makes unreportable.

**The ready set is one shared intrusive list under a mutex.** Intrusive
because the link lives in the actor, so an enqueue allocates nothing and
cannot fail, which the wake protocol requires: a unit in `Woken` is owed a
slot, and an enqueue with a failure path would have nowhere to record the
debt. A mutex rather than a lock-free structure because BEAM tried lock-free
run queues, kept one mutex per queue, and recorded that the lock-free
version performed worse without the reason being investigated. The rate that
reaches it is one enqueue per newly-ready actor and one acquisition per
batch refill, not one per operation.

**Each worker has two private lists, single-owner and without atomics:**
pinned ordinary coroutines, and units woken mid-message. Both are intrusive
for the same reason as the ready set. A worker drains its own lists first,
because that work is warmest, alternating between them under a budget of
consecutive items so that an actor is not left behind a stream of pinned
coroutines that park and re-wake on their own thread.

**The budget counts items and can bound nothing else.** Duration is beyond
it: this substrate has exactly two preemption points, a park and a message
boundary, and a pinned coroutine computing for two hundred milliseconds
reaches neither. Everything queued behind it on that worker waits, and no
other worker may take it — the coroutine cannot move at all, and nothing in a
private list is readable from outside. That is not mitigated, it is the shape
of a runtime without preemption, and it is the reason a batch taken from the
ready set stays small: work in a batch is as trapped as work in a private
list, so a batch is a prefetch rather than a reservation. Both the batch and
what is mounted count as load, or placement keeps sending work to the worker
that is deepest in it.

With its lists empty a worker refills its batch from the ready set in one
acquisition, preferring actors whose last worker was itself, because their
arena is warm there. **The preference is a bounded look and not a search:**
it examines a fixed prefix and takes the head otherwise. A scan proportional
to the list's length would put the wrong thing inside the one shared critical
section, since that list is longest exactly when the machine is busiest.

**A woken mid-message unit goes to its owner's private list, and leaves it
only under the opt-in.** The default is the price rule of the entry above: a
move here carries the live stack and the saved registers, while a move
between messages carries nothing. It goes to the ready set instead only when
four things hold together — the actor declared mid-message movement, the
owner is loaded, the unit's foreign-frame counter is zero, and this actor has
not already moved during this message. The last condition is Edmond's, and it
is what keeps the price bounded: the profile this exists for parks a hundred
times inside one message, and moving at every park would take the ready
set's mutex a hundred times where one message should take it once.

Either branch can be wrong and neither is revisited. A unit sent to the ready
set whose owner frees up a moment later paid the move for nothing; one left
with a loaded owner waits for it. One move per message bounds the error as
well as the price, and none of it is measured.

**No worker reads another worker's list, and there is no thief.** An idle
worker looks in exactly one shared place. Rejected on this axis: a shed
lane per owner, which would force a worker with empty lists to read N−1
foreign lanes before it may sleep, restoring an O(N) foreign read per idle
transition and the O(N²) fleet behaviour that disqualified stealing in the
first place — Go's own diagnosis at 56 cores is that "the work stealing
algorithm degenerates to O(N²) due to N cores all inspecting each other's
mostly empty run queues" (golang/go#28808, open since 2018).

**"This worker has nothing to do" is one predicate over five places** — its
two private lists, its batch, its intake queue and the ready set — and it is
written once, in one function. Go's `runqempty` went silently wrong the day
`runnext` was added, and every balancing decision built on it inherited the
error. The intake queue belongs in the predicate: a worker that sleeps on an
undrained intake queue sleeps on a conditional resolution the collector is
waiting to have applied.

**The last worker to sleep still owes `design/deadlock.md` its observation**,
and that document asks for "every run queue empty" from a time when there was
one queue per worker. Restated for this structure: the last worker is the one
whose idle bit completes the mask, and it reads the ready set's emptiness
under the mutex it is about to release. No other worker's private lists have
to be read, because a worker holding anything in its lists has not published
its bit. The condition may be read racily, as that document allows, since a
needless pass is only a pass.

**Load is two numbers.** Placement reads total load, pinned coroutines
included, because a worker deep in work that cannot move must stop
receiving actors that could have run elsewhere. Shedding and the drain
budget read the actor class alone. Both count what is mounted beside what is
queued, for the reason erlang/otp#9762 records.

**An enqueue into the ready set is what rings a sleeper, and it rings
exactly one.** The insertion reports whether the set was empty before it,
and only that edge wakes anybody — the mechanism TrueAsync's cross-thread
queues use, where the enqueue hands back `was_empty` and the caller signals
on it alone. The signal is the wake descriptor the reactor already has
(`design/reactor.md`), and it is rung outside the ready set's mutex, because
a system call inside the one shared critical section would serialize every
worker behind it. A sender may be a thread that is no worker at all — a
consumer calling in over the C ABI — so the obligation belongs to the
enqueue rather than to any worker's turn.

**Going to sleep is an order, not a check.** A worker publishes its idle
bit, then re-reads the ready set, and only then sleeps; without that order
an enqueue that lands between the check and the sleep rings nobody, since
the bit was not yet visible. The bit says asleep or about to be, and a
worker woken for work clears its own bit — a waker that cleared it would
leave a worker that never woke invisible to the last-to-sleep condition
below.

**The collector roots the memory mark in the scheduler's ownership table and
in no queue.** Ruled by Sage, and it repairs what this entry first said. The
table is a registry of lifetimes rather than a map of current owners: a unit
or an actor joins the creating worker's shard when it is created and leaves on
its terminal transition, while a move never touches the table, because the
current owner is named by the claim word (see the re-mounting entry of this
date). Rooting in the queues instead would have the collector read
non-atomic single-owner lists from its own thread, which it cannot do without
a race. The prohibition in `design/deadlock.md` is untouched: it forbids the
scheduler's table as a *liveness* root, and this is the memory mark. L is
rooted nowhere in the scheduler.

Four invariants make the table complete, each checkable at a queue operation:

1. only a registered handle is ever pushed;
2. an object's word takes its queued value before the push — `ready` before
   the ready set, `Woken` before a private list — which both protocols
   already prescribe;
3. a pop does not remove from the table; removal happens on the terminal
   transition alone, after which no enqueue of that handle is possible;
4. scheduler queues are intrusive and carry no payload: nothing lives only in
   a queue.

Together they give "reachable from a scheduler queue implies registered", so
the walk begins with complete roots without reading the ready set, the private
lists, a batch, or an in-transit cell. What the collector asks about an actor
— whether a unit is mounted, whether one is queued, whether the mailbox is
empty — it reads from the actor's own words, and invariant 2 is what makes
those reads truthful. The cost is one shard lock at registration and one at
removal, once in an object's life, plus a root scan the size of the tables. The handshake is narrower than the roots. A worker answers only for actors
at a message boundary — idle, in the ready set, or in its batch — because
only there is the arena consistent. A unit in the woken-mid-message list was
unmounted with reason `Park`, so its arena holds whatever the message left
half-done, and `design/execution.md` forbids answering for it. Mark
termination waits on such a unit exactly as it already waits on a parked
actor: a cost that document records, not a new one.

**The message rides the mailbox; only the placement rides a queue. What
this obliges `design/deadlock.md` to change is its actor-call row.** That row
reads the callee's current unit, a queued unit, or a non-empty mailbox, and
all three are false in one window: the worker has taken the message out of
the mailbox and has not yet created the unit. Readiness now lives in the
actor's readiness word, so the row has to read it, with ready or running
counting as live. Without that a healthy synchronous call between two actors
is reported deadlocked, which breaks the one promise the detector rests on.
An earlier form of this entry asked instead that a mailbox deposit become a
served-set form; that was the wrong repair, because a sender writes the
mailbox directly and rides no intake queue, and the case it named was already
covered by the non-empty-mailbox branch.

**Rejected, with the property each lost on.** `crossbeam-deque`: it drags in
`crossbeam-epoch`, a second deferred-reclamation machine beside the
quiescence scheme in `design/pool.md`; it allocates on the push path when
the buffer is full and on the pop path when it shrinks below a quarter; its
buffer is a private `Atomic<Buffer<T>>` with `Worker<T>` not `Sync`, so no
third thread may enumerate it for rooting; and its steal takes whatever is
at the far end, which cannot express a refusal. A thief as the primary
mechanism: the foreign reads above, and a hit rate worse than Go's, because
most of what it would find is pinned by construction. One global queue with
no private half: contention on every mount rather than once per batch.
A bounded ring with a shared index for either structure: TrueAsync's own
benchmark measured that shape at 2.3 M ops/s against 56–68 M for
per-producer sub-queues at eight producers.

**Unmeasured, and named as such:** the batch size, the drain budget, how
many messages of one actor a worker runs before returning it to the ready
set, whether the ready set wants splitting per NUMA node, and what a cold
arena is worth in the take preference. Nothing here is measured on our
workload, because there is no code. The figure that decides where
measurement starts is the one above: a per-message placement rate
approaching a million per second is within an order of magnitude of the
contention collapse that benchmark found, so the ready set is the first
thing to put under a load generator. Two acquisitions per message is the
worst case — one placement and at most one mid-message move — and that holds
only because of the one-move bound. Without it the profile the opt-in exists
for, a hundred parks in one message, would take the shared mutex a hundred
times per message, and the collapse would arrive four orders of magnitude
earlier than this paragraph implies.

## 2026-08-13 — Superseded: two run queues, placement instead of stealing

The entry this replaced proposed placement at wake time with surplus shed by
the owner into a per-worker lane. A Critic round found nine high-severity
defects; four were structural and are what the entry above answers. Kept
here as the record of what was tried: an idle actor has no unit, so nothing
arbitrated its placement and neither the "exactly one enqueue" invariant nor
the foreign-frame counter applied to it; the vetting that justified
owner-side shedding was available only where the decision was already local;
per-owner lanes restored the very scan that disqualified stealing; and
filling a lane at a turn boundary tied the offload rate to the boundary rate
of the busiest worker, which is also why "bounded by one message" was not a
bound.


## 2026-08-13 — The waiter cell: what a pool slot names, and who empties it

Applying the retirement of the 64-bit handle to `design/pool.md` exposed a
question the corrections table never asked: an operation slot used to name
its waiter by a handle whose generation rejected a stale read, and a
coroutine has no generation now. Sage's ruling, executed as written.

**A slot names its waiter through a waiter cell** — a counted reference to
the unit, the entry index and the epoch — and the reference is what keeps
the unit alive across the window. The cell is written at step 2 of the
parking protocol by the unit's own thread, which owns the ring the
operation goes to, and emptied by that same worker at whichever comes
first: the completion carrying the result, which moves the reference into
the `wake` it calls, or the retire of the entry, which drops it before the
kernel cancel is submitted. One thread performs both, so a completion never
loads a reference another thread is dropping. A completion that finds the
cell empty adjusts the owed count and wakes nobody.

**Two slots hold no cell of their own.** A multishot operation names the
socket's handle and never a unit, because the unit re-parks between chunks;
the reference for a wait on that queue lives in the socket slot's own
queue-waiter cell. A cancel operation names the operation it cancels.

**The waiter cells of armed slots are memory-mark roots**, alongside the
timer wheel and the reactor intake queues. Rejected: the argument that the
cell is safe because a unit with an armed entry is parked and therefore
reachable from the scheduler's ownership table. It fails for a window — a
retire posted across threads is applied a turn later, and inside that turn
the unit can resume, finish and leave the table while the cell holds what
may be the last reference — and it fails on the model, whose collector
explains a reference count by enumerated in-edges: a counted cell the walk
cannot see leaves the difference permanently positive and reports the
runtime's own pools as external roots on every pass. L is rooted in no pool
and flows through none, so the liveness fixpoint still walks nothing there.

**A cross-thread cancel request carries a promise**, resolved by the one
worker that applies it, which reads the state word on the thread that owns
it. The requester reads no state word. Same-thread, the answer is the
call's return. For an actor that declared mid-message movement two answers
come from the delivery rule's single load: `Terminal` is *already
finished*, `WokenShared` is *delivered*.

**A cancel for a unit in `WokenShared` is the cancelled byte**, written
from wherever the canceller is; the worker that wins the mount inherits the
bit. `design/execution.md`'s step 0 had compressed the delivery rule to
"forward or drop" and lost that case.

Two defects of mine that the same round found, both now fixed in the
documents. A cancel decides a wait by claiming the winner field, and the
record therefore carries that field under AND as well as under OR: nothing
else could represent "decided" once the epoch stopped being bumped, and a
canceller writing `remaining` to zero would be indistinguishable from the
last entry arriving while every entry was still armed. And `Terminal` is a
value of the state word rather than a fourth parking state, stored after
the unit's own code has finished and before the wait record is released —
earlier reports an ordinary unwind as a deadlock, later lets a retired
completion read a freed record — with a terminated coroutine excluded from
the liveness roots, since it exists only because something still references
it.

## 2026-08-13 — Deciding a wait without an entry, and three protocol holes it opened

A second Critic round against the corrections of S5.3 found four defects in
what the first round's fixes had produced. Sage's ruling, executed as
written.

**A wait decided by no entry stores its error in the record's own result
slot.** The record carries that slot beside the winner field, and a cancel
or a detector resolution stores there and then claims the winner with a
reserved value no entry index takes. An entry's result slot keeps exactly
one writer — the wake that ends that entry — so a retired entry firing late
writes where nobody reads. Without the slot the cancellation had nowhere to
live but an entry, and step 2 of the wake protocol overwrote it: a channel
send arriving after the cancel replaced the cancellation with a value, and
the unit resumed reading data it was told it would not get, final
cancellation included. **Step 2 does not move behind step 3**, and the
reason it stands first is the AND return: a caller that does not take
`remaining` to zero returns at step 3, so its result must already be
stored.

**A cancel against `WokenShared` writes the byte, re-reads the word, and
answers from the re-read.** Supersedes the single-load clause of the waiter
cell entry above, which let the canceller answer *delivered* on the strength
of one load. One load orders nothing against the mount: the winner can mount,
run, park and read an unwritten byte, and the byte lands where nothing reads
it again — a shutdown counting a *delivered* against a unit it then waits on
forever. The byte's store, the re-read, the mounting compare-exchange and
park step 4's read of the byte are sequentially consistent for a declared
actor, so a re-read that still says `WokenShared` puts the byte ahead of the
mount. Rejected: folding the bit back into the word, which would make every
owner-side transition a compare-exchange loop — the cost the byte exists to
avoid.

**A wait on a socket's multishot completion queue is always live**, and the
kind stays external and not cycle-capable. Calling it dead while the series
is un-armed would require proving that no buffer ever frees, which is false
in the routine `-ENOBUFS` gap and would invent a deadlock against a loaded
healthy connection. The one state the row cannot report — a waiter over a
queue the reactor can never re-arm — is a fact the reactor already holds, so
the reactor publishes it from a re-arm list after a threshold, with the
buffer pool's state and a starved-re-arm counter.

**The park-time recheck of that queue belongs to whoever writes the waiter
cell, immediately after the write.** The ring's own worker writes and
rechecks inline; a unit on another worker posts the publish and suspends
without reading the queue, and the owner rechecks when it drains the entry.
The unit's own remote re-read defended nothing, because it ran before the
publish it was meant to defend: a chunk appended in between found an empty
cell, and the unit slept over a full queue until an unrelated chunk arrived,
which for an idle socket is never.

Named and not ruled: the close path owes the sentence that closing a socket
empties the queue-waiter cell into a wake carrying the close error, which the
always-live row rests on. Written into `design/reactor.md` with this entry.

## 2026-08-13 — No pool is a memory root, L stops at a debtor's owner field, and the generation stands at 32 bits

Sage's second ruling of the day, on three defects the Critic found in the
application of the waiter-cell entry above. Executed as written.

**Struck from that entry: "the waiter cells of armed slots are memory-mark
roots".** A root whose enumeration is a slab walk is not a root, because
that walk's own first rule is that it may miss a slot: a scan passing a free
slot that is taken a moment later never sees the reference written into it,
and a unit whose cell holds the last reference is freed under a retire still
queued. **No pool is a root, and none needs to be.** While a wait is
undecided the scheduler's ownership table reaches the unit, since a unit
leaves that table only on its terminal transition and reaches one only by
resuming, which requires the wait decided. Once it is decided, every cell is
emptied inline on the unit's own thread except a socket's on another worker,
and that pending retire is an intake entry — already a root with a defined
scan. The pass therefore costs the pools nothing, by either mark, and the
earlier claim that the scan was free because the collector would run it
anyway is struck as false.

**A cross-thread retire drops no reference where it runs.** The request
carries a counted reference the poster took on the unit's thread; the
applier moves the cell's reference into the confirmation it posts back; both
are dropped on the unit's thread. Without it a foreign worker decremented a
non-atomic count — a defect the waiter-cell entry introduced and the Critic
did not raise.

**L does not flow through the fields by which a debtor names its debtor** —
a mutex's holder, a guard-semaphore's holder list, an actor's current unit,
a join's target — and the attribution walk crosses none of them. The
retirement of the 64-bit handle left a counted reference as the only way to
name a coroutine, and a counted reference conducts marks where a handle
conducted none: a mutex reachable from any global marked its holder live,
the waiter's row read "the holder is L-marked", and the row was satisfied by
its own premise, so no mutex cycle was reportable on any pass. The field
stays a counted reference, because the liveness rows read a *terminated*
debtor out of it and the corpse must remain readable. `design/deadlock.md`
owns the sentence, being the only reader of the field.

**Thirty-two bits of generation stand, and the epoch argument that defended
them is struck as circular** — the epoch a completion validates comes out of
the slot itself, in this design and in the one before it, so it validated
the current occupant against itself. What closes the wrap is a bound the
corpus already contained: a slot still owed completions does not release,
each reuse after release requires every worker to pass a quiescent point,
and every comparison the substrate performs happens at the drain of the
reading worker's current or next turn. The window admits a few hundred
reuses where a false pass needs 2³². **This makes "no pool handle crosses
the API" a load-bearing rule**, stated now rather than assumed; a surface
that hands one out reopens the width question for that pool alone. The
"generation width" open question is deleted.

Left open by name: whether the collector's barrier discipline covers a
counted reference handing off between two roots mid-pass, which joins the
`ll-model` integration item rather than being decided here.

## 2026-08-13 — Toolchain: one pinned rustc, and its LLVM is the project's

`rfc/runtime/implementation-language.md` makes this a build-system rule
rather than a preference: every participant that produces bitcode must
share one LLVM version, and since rustc pins its own, **rustc's LLVM
dictates the project's**. The C++ layer, the JIT and Clang are built
against it. That single sentence decides most of what follows.

**The toolchain is pinned in `rust-toolchain.toml`, not bounded by a
minimum.** A range is what a published library offers its consumers; this
substrate is compiled into one artifact beside a C++ layer that must hold
the same LLVM, so a consumer free to choose an older rustc is a consumer
free to break the link step. There is therefore no MSRV distinct from the
pinned version: they are the same number, and raising it is an edit of one
file plus a rebuild of the C++ layer.

**Pinned today: rustc 1.96.0** (`ac68faa20`, 2026-05-25), which reports
LLVM 22.1.2. Verified by running it rather than read from a table.
Cargo 1.96.0 ships with it. **Edition 2024**, which is what that toolchain
emits for a new crate.

**Two version checks belong in CI, and both fail on the machine this was
written on.** `llvm-config` here reports 22.1.8 against rustc's 22.1.2 —
same minor, different patch, and the rule as written says one version, so
whether a patch difference is admissible has to be decided before the
first bitcode is linked rather than discovered at link time. The system
Clang here is 18.1.3, four major versions below, and cannot participate at
all. Neither figure is a property of the project; both are what a
developer machine looks like without the pin applied, which is the
argument for checking it in CI instead of in a README.

**Backends are cargo features, one per kernel interface**: `io_uring`,
`iocp`, `kqueue`, `epoll` (`design/reactor.md`). A target enables the
features its kernel offers, and a consumer that runs computation without
I/O compiles none of them.

**CI targets follow the platform table of `design/switching.md`**, because
that table is what the context switch is written against and an untested
row there is a row nobody has run:

| Target | Switch row | Backend |
|---|---|---|
| `x86_64-unknown-linux-gnu` | x86-64 SysV | `io_uring`, `epoll` |
| `aarch64-unknown-linux-gnu` | AArch64 ELF | `io_uring`, `epoll` |
| `x86_64-pc-windows-msvc` | x86-64 Windows | `iocp` |
| `aarch64-pc-windows-msvc` | AArch64 Windows | `iocp` |
| `aarch64-apple-darwin` | AArch64 Darwin | `kqueue` |
| `armv7-unknown-linux-gnueabihf` | ARM 32-bit | `epoll` |
| `riscv64gc-unknown-linux-gnu` | RISC-V 64 | `epoll` |

The first three gate a merge and run their tests; the rest are built and
cross-checked without running, until there is hardware or an emulator
worth the minutes. Which of them ever gets a running test is a question
for the stage that writes code, and it is open.

Cost: a pinned toolchain means a Rust release the project cannot take
until the C++ layer follows it, so the upgrade is one change to three
things at once. That is the price of linking bitcode from two compilers,
and the alternative — letting rustc float — buys nothing, because the
LLVM it drags along is not ours to choose.

## 2026-08-13 — Module boundaries: the intake queue, the cut through park and wake, and a ninth module

Supersedes the module list of the crate entry of 2026-08-12, which named
`io-core` as unit, switch, stack, sched, pool and deadlock. It is now unit,
switch, stack, sched, pool, **sync**, deadlock and the unit half of cancel.
Sage's ruling on three boundary questions a Critic round raised against
`dev/ARCHITECTURE.md`; executed as written.

**The intake queue belongs to `sched`, whole, in `io-core`.** Five of its
six users are `io-core` modules, its element vocabulary is `io-core`'s
throughout — unit references, epochs, entry indices, resources, promises,
resolutions — and `sched` drains it every turn while the nothing-to-do
predicate reads its emptiness and the detector's pass enumerates its
entries. A structure whose type, drain, predicate, scan and roots are all
`io-core`'s is an `io-core` structure. Rejected: reactor ownership with an
installed post callback, on the shape of the cancel handle. A callback is a
blind call, and this queue is not blindly called.

**An entry is a declaration and an applier.** The declaration is what any
reader sees without knowing the poster: the counted cells the entry holds,
which is what makes the queue a memory root; the unit and epoch it names,
or none, which seeds the liveness mark; the resource it would move, or
none, which feeds the served set — and that field is the same field a
forwarded supply wake's payload already carries, not a second one. The
applier is installed by the poster and run at the drain, in arrival order,
with `sched` looking behind none of them.

**What crosses the crate line instead is the worker driver.** `sched`
defines it — flush submissions, drain completions, sleep until the waker
rings, arm and remove a timer-wheel entry — and `io-reactor` implements it
per worker. Ringing a sleeper goes through its waker. With the cancel
handle and `sync`'s duty hook this makes three crossings of one shape: the
module that knows installs an opaque callable, and the module that holds
the boundary invokes it. The detector's watchdog reaches the timer wheel
the same way, or it is the forbidden edge under another name.

**The ordering contract of the intake queue is `sched`'s debt now**, not
`design/reactor.md`'s: one queue per worker, drained in arrival order
across producers, with resolutions and deposits in the same queue, because
`design/deadlock.md`'s step 12 and the served set rest on it. What the
structure costs is the same open question it was.

**`park` and `wake` are `sched`'s exports; `unit` keeps the record
mechanics.** `design/execution.md` still specifies both protocols whole,
and the split is by step. Dispatch, the enqueue, the move path with its
four conditions, and the ring of a sleeper are `sched`'s. The epoch test,
the result store, the decidedness test, the claim, retirement through
opaque handles and the suspension itself are `unit`'s. Each entry is armed
by its resource's module, which installs the cancel handle. The owner half
of a declaring actor's wider word is opaque to `unit`: it stores the tag
`sched` passes and resolves nothing, which is how it carries `Running(W)`
without naming a worker. Rejected: inverting the two, so that `unit` sits
above `sched`. The mounting compare-exchange, the ready-set entry's
disambiguation and the delivery rule are written in the state word's
vocabulary; inverting would hide that vocabulary behind a trait and move
no knowledge at all.

**`sync` is a module of its own because every candidate absorber is
forbidden a piece of it**: `unit` must not know what a channel is, `sched`
must not either, S5.3 severed these objects from `pool`, and `deadlock`
reads observables and nothing behind them. It is in `io-core` because the
detector is, and reads occupancy, both end counts and the flags off these
resources; and because a consumer that compiles no backend still gets
channels, futures, mutexes, joins and semaphores. The debtor's owner field
is `sync`'s to keep truthful and `deadlock`'s to mark: the memory mark
traces it, the liveness mark stops at it, the attribution walk does not
cross it. A design document for the mutex, the join and the semaphore is
still owed; the actor mailbox stays the open row it is in
`design/deadlock.md` and `sync` does not take it.

Struck by this entry: `design/channels.md`'s "the intake queue itself is
`design/reactor.md`'s"; `design/reactor.md`'s claim to owe that queue's
ordering contract; the same debtor in `design/deadlock.md`'s open questions
and in `dev/INDEX.md`. Every "reactor intake queue" in the design documents
is a worker intake queue; superseded entries here keep their wording.

## 2026-08-13 — Project Loom as a precedent: what it closes and what it lends

Recorded because Loom made the opposite trade on our central question and
shipped it, so its experience is evidence rather than opinion. Nothing here
changes a decision; two items become obligations and one becomes an open
question.

**The heap-stored stack is closed to us, and not for want of ingenuity.**
Loom stores a suspended virtual thread's frames in a heap object and copies
them back one frame at a time, patching a return address so that the next
return traps into the runtime and thaws the next frame. Both halves require
the runtime to have compiled every frame in the stack: it must know each
self-referential pointer to rewrite when the object moves, and it must be
free to overwrite a return address. A C ABI supplies neither. The Loom
proposal of 2018 names that as the precondition it chose to accept —
supporting native languages would require the stack to be contiguous and to
stay at one address, and a language runtime is not obliged to support
arbitrary native code. The shipped record agrees: JEP 444 named two cases
that pin a virtual thread to its carrier, `synchronized` and native code;
JEP 491 removed the first in JDK 24 and kept the second deliberately. Eight
years and a dedicated JEP did not close the foreign-frame case. Treat it as
settled rather than open.

**Three findings transfer, and none is about stacks in the heap.**

*Work stealing has a measured failure with a profile.* JDK-8360046, Sergey
Kuksenko, Oracle, 2025-06-19, on a two-socket Xeon 8368 of 38 cores each:
virtual threads diverge from platform threads at 16 cores and are 4.4x
slower at 32 on the same work, with one instruction —
`compareAndExchangeCtl` inside `ForkJoinPool::signalWork` — taking 17.12%
of all cycles against 1.78% for the platform-thread case. The repair landed
in JDK 26 and was backed out twice, JDK-8375130 and JDK-8379525, because it
traded that contention for starvation. Their machine and their workload,
not ours. What it obliges: our enqueue into the shared ready set and the
ring of a sleeper is the same shape, and when it is measured it is measured
against that number rather than against the O(N²) scan we rejected stealing
for. The two are different mechanisms and this is a second, independent
reason.

*Enumerability and collection are in direct conflict, and the JDK shipped
the wrong default for four major versions.* JEP 425 states that a parked
virtual thread nobody can reach is collectable. It is not: `ThreadContainers`
keeps every started virtual thread in a static set of strong references
unless told otherwise. Measured by an agent of mine on this machine — WSL2,
16 cores, OpenJDK 21.0.2, SerialGC, 200 000 parked virtual threads at depth
one — 741.9 bytes retained per thread whether the program held references or
dropped them, and 0.6 bytes per thread with `-Djdk.trackAllThreads=false`.
Measured on Java, not on us. What it obliges: our detector needs coroutines
enumerable and our scheduler's ownership table is a memory root, so whether
that table retains a coroutine that can never run again is a question this
corpus has not asked. **Open.**

*Never fix a stack's size class from its first observation.* The Quarkus
team instrumented the JVM under Netty with ten thousand connections and
measured a reused stack chunk of 1149 machine words against a steady-state
live stack of about 286 — a fourfold overshoot, because the capacity is
fixed at the first freeze, which happens during warm-up while the same code
is still compiled by the first-tier compiler and produces larger frames.
Their machine and workload. What it obliges: `design/stacks.md`'s size
classes and warm watermark must not be chosen from a first observation.

**Two positions of Loom's confirm ours and add nothing.** Ron Pressler
refused to compensate for pinning by creating carriers, on the grounds that
pinning has no bound and new carriers pin too, preferring "a deadlock and
observability reporting that can help the programmer actually solve the
issue" (JDK-8334304, 2024-07-08) — which is `design/deadlock.md`'s thesis.
And forced preemption was removed from Loom outright, `openjdk/loom` commit
`8a4613775b74` of 2022-04-02, leaving only the VM-internal path; our
"a coroutine always ends itself" is the same position.

**One counter-example is worth keeping.** `ForkJoinPool`'s saturation
predicate lets the pool continue with fewer threads than its target when
compensation would exceed the maximum, and its own documentation says this
"might not ensure progress". Silent degradation where we chose a report:
the reactor publishes re-arm starvation rather than quietly serving fewer
sockets.
