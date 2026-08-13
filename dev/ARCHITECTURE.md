# Architecture — knowledge map

Who knows what, and what each module must not know. Written before the
code, and rewritten whenever a boundary moves rather than after.

Most modules below have a design document behind them (`INDEX.md`); three
do not, and `INDEX.md` records which. Where this file and a document
disagree, the document holds; where a document and `DECISIONS.md`
disagree, `DECISIONS.md` holds. The boundaries themselves were ruled on
2026-08-13, and that entry is the authority for them.

## Crates

One repository, one cargo workspace, three crates. The split follows the
dependency direction, so a consumer that runs computation without I/O
compiles none of the platform backends — and still gets channels, futures,
mutexes, joins and semaphores, which is why those live in the core crate.

| Crate | Holds | Depends on |
|---|---|---|
| `io-core` | `coroutine`, `switch`, `stack`, `sched`, `pool`, `sync`, `deadlock`, the coroutine half of `cancel` | memory manager |
| `io-reactor` | `reactor` and the four backends, each behind a cargo feature; the operation half of `cancel`; the worker driver's implementation | `io-core` |
| `io-api` | the public surface and the C ABI | `io-core`, `io-reactor` |

`io-reactor` knows `io-core`; the reverse dependency does not exist and
must not appear.

**Three things cross that line, and all three take the same shape: the
module that knows installs an opaque callable, and the module that holds
the boundary invokes it without looking behind it.**

- **The cancel handle.** A cancel is delivered through the state word, not
  through an entry (`design/cancellation.md`); each entry is retired
  through the handle its armer installed. One kind of armer lives in
  `io-reactor`, so that crate supplies a handle that submits the
  kernel-side cancel and waits out what the kernel still owes.
- **The worker driver.** `sched` defines what a worker's turn needs —
  flush submissions, drain completions, sleep until the waker rings, arm
  and remove a timer-wheel entry — and `io-reactor` implements it per
  worker. Ringing a sleeper, from an enqueue or from a post, goes through
  its waker.
- **The duty hook.** A supply wake refused on another worker has to be
  re-run against the resource (`design/channels.md`); `sync` installs a
  hook, and `sched`'s drain invokes it.

**The intake queue does not cross the line.** It is `sched`'s, whole, in
`io-core`: one queue per worker, and **every signal addressed to a coroutine
that starts on another thread travels through it**. Three crossings do
not: an enqueue into the ready set, a push into an actor's mailbox, and
the cancelled byte of a declared actor. Each addresses an actor rather
than a coroutine, and each carries an ordering rule of its own.

An entry is a **declaration** and an **applier**. The declaration is what
any reader may see without knowing the poster — the counted cells the
entry holds, which make the queue a memory root; the coroutine and epoch it
names, or none, which seed the liveness mark; the resource it would move,
or none, which feeds the served set. That last field is a word `sched`
carries and never interprets: the detector resolves it against the
resource kinds at its own step, which is how the queue stays free of any
resource type.

The applier is installed by the posting module and run by the drain, in
arrival order, with `sched` looking behind none of them.

## Modules

Ten rows, nine modules and the API crate, each with the same six lines.
Files are proposed, not written; naming them here is what this stage is
for.

### `coroutine` — one resumable flow of control

- **Files:** `coroutine/mod.rs`, `coroutine/record.rs`, `coroutine/state.rs`,
  `coroutine/stackful.rs`, `coroutine/stackless.rs`
- **Public:** `Coroutine`, `CoroutineRef`, `State`, `WaitRecord`, `Entry`,
  `EntryIndex`, `Epoch`, `WaitMode`, `CancelHandle` — which it defines and
  every armer installs into — and the record operations: write the record,
  test the epoch, apply a result, test decidedness, claim the winner,
  retire the entries, store the state word
- **Crosses its boundary:** a counted reference, which is the only name a
  coroutine has; the record operations, which `sched` composes into the
  callable `park` and `wake`; the `walk` hook by which the memory manager
  reaches its cells — the two inline entries and the spill block — which is
  what the detector rides; the joiner list it wakes at teardown, after
  storing `Terminal` and before releasing the record
- **Knows:** the two suspension kinds and the ABI a compiler owes for the
  stackless one; the record — per entry a resource, a cancel handle, a
  result slot and a fired bit, and once per record the mode, `remaining`,
  the winner field, the record's own result slot and the epoch; the state
  word, including the wider word a declaring actor carries; the record
  steps of both protocols and the suspension itself
- **Does not know:** which backend completed an operation, what a socket
  is, what a channel is, which worker owns it — the owner half of its own
  wider word is a tag it stores and never resolves
- **Depends on:** `stack`, `switch`, memory manager — a coroutine is an ordinary
  entity of it, with a class the runtime builds and a spill block freed
  through deferred release

### `switch` — context switches

- **Files:** `switch/mod.rs`, `switch/x86_64_sysv.rs`,
  `switch/x86_64_win.rs`, `switch/aarch64.rs`, `switch/arm.rs`,
  `switch/riscv64.rs`, `switch/narrow.rs`, `switch/foreign.rs`,
  `switch/cet.rs`
- **Public:** `switch_to`, `SavedContext`, `ForeignFrames`
- **Crosses its boundary:** the saved context, which this module alone
  lays out and which sits in a region at the top of a stack that `stack`
  lends without knowing its shape; the foreign-frame counter, and it has
  three readers — `sched` before a re-mount, `cancel` before it answers
  *pinned* or *not deliverable*, and `deadlock` while it walks
- **Knows:** the preserved set of each of the seven platform rows, the TEB
  and shadow-stack swaps, the suspendable calling convention, when a
  switch may narrow
- **Does not know:** scheduling policy, I/O, actors, the coroutine
  object's layout
- **Depends on:** `stack`

### `stack` — stack memory

- **Files:** `stack/mod.rs`, `stack/classes.rs`, `stack/reserve.rs`,
  `stack/pool.rs`, `stack/guard.rs`, `stack/shadow.rs`,
  `stack/overflow.rs`
- **Public:** `Stack`, `StackPool`, `SizeClass`, `ShadowStack`, `take`,
  `release`
- **Crosses its boundary:** a stack lent to a coroutine for its lifetime; the
  region at its top that `switch` lays out; the size hint a consumer may
  pass over the C ABI; the fact that a coroutine may enter foreign code, which
  is the one property of a coroutine this module reads, because such a coroutine is
  refused a slab-carved stack and gets its own mapping with a guard page
- **Knows:** reservation and lazy commit per platform, size classes and
  the warm watermark, guard bands, the mapping ceiling and the two ways
  past it, what release means on each platform, and shadow stacks —
  allocated by their own call, pooled by remapping, sized by depth against
  a class table of their own; overflow detection, which is a per-platform
  opt-in with a signal handler, an alternate stack and a Windows structured
  exception path
- **Does not know:** what runs on a stack, beyond whether it may enter
  foreign code
- **Depends on:** memory manager

### `sched` — mounting coroutines on threads

One module and three parts, because the predicate that decides a worker
has nothing to do reads all five places at once and is written once.

**The worker.** The turn, the message budget, the mount and unmount hooks
with the consumer's actor context, the quiescent-point counter it
publishes for `pool`, the drain of the intake queue, and the driver it
defines for `io-reactor` to implement.

**Two private lists per worker** — pinned ordinary coroutines, and coroutines
woken mid-message. Single-owner and without atomics, because an ordinary
coroutine's reference counts are non-atomic and it is pinned for life, so
**no other worker may read them**.

**The shared ready set** — one intrusive list under a mutex, one entry per
ready actor and never a message, plus the actor's atomic readiness word,
the idle mask, and the batch a worker prefetches under one acquisition.
**Any thread may enqueue here**, which is the opposite rule and the reason
these two parts cannot share a paragraph.

- **Files:** `sched/mod.rs`, `sched/worker.rs`, `sched/driver.rs`,
  `sched/local.rs`, `sched/ready.rs`, `sched/intake.rs`,
  `sched/readiness.rs`, `sched/mount.rs`, `sched/ownership.rs`
- **Public:** `Scheduler`, `Worker`, `WorkerId`, `Readiness`, `Intake`,
  `IntakeEntry`, `WorkerDriver`, `WorkerWaker`, `MountHook`,
  `UnmountHook`, `park`, `wake`, `enqueue`, `spawn`
- **Crosses its boundary:** the ownership table, a memory root and a
  registry of lifetimes, which is what reaches a coroutine whose wait is still
  undecided (`design/deadlock.md`); the
  intake queue, the other root and the one path a payload takes across
  threads; the actor's readiness word, which the detector reads; the
  quiescent-point counter; the worker driver and its waker; the actor
  handle, which carries the substrate's single `unsafe impl Send` with the
  foreign-frame counter as its precondition
- **Knows:** the five places and the one predicate over them; wake
  dispatch and the delivery rule; the enqueue, and the move path with its
  four conditions; the transitions of the readiness word; that a sleeper
  is rung through the driver's waker; that a park may have to arm a
  watchdog and a last-to-sleep worker to request a pass, both of which it
  performs through predicates the detector installs
- **Does not know:** how a coroutine suspends internally, how an operation
  completes, what a channel is, which waits are worth a watchdog, what is
  behind an applier, a duty hook, a driver or an arming predicate
- **Depends on:** `coroutine`, `switch`

### `pool` — slabs, slots and the waiter cell

- **Files:** `pool/mod.rs`, `pool/slab.rs`, `pool/handle.rs`,
  `pool/waiter.rs`, `pool/operation.rs`, `pool/socket.rs`,
  `pool/timer.rs`, `pool/buffer.rs`, `pool/walk.rs`
- **Public:** `Handle`, `Pool`, `Slot`, `WaiterCell`, `Walk`, `take`,
  `release`, `resolve`
- **Crosses its boundary:** handles, and **only inside the substrate** —
  no pool handle reaches `io-api`, which is what bounds the staleness
  window a 32-bit generation has to survive; the `kind` byte of a slot
  header, which the detector resolves a handle to read; the
  quiescent-point counter it consumes from `sched`; the walk itself, which
  is what the diagnostic dump is composed from and is not the dump
- **Knows:** slabs that never move and never free; the generation,
  deferred reclamation, and the bounded window that makes a 32-bit
  generation safe; the waiter cell's one-thread rule; what a validated
  read of a slot header does and does not guarantee, and that the dump
  accepts false negatives, re-validates any conclusion spanning several
  slots, and never follows a waiter cell
- **Does not know:** the layout of a coroutine or an actor — neither is a
  slot; what a wait edge means; the reactor backend
- **Depends on:** memory manager, `coroutine` (a waiter cell holds a counted
  reference and an entry index), `sched` (the quiescent point)

### `sync` — channels, futures, and the primitives a wait can name

- **Files:** `sync/mod.rs`, `sync/channel_local.rs`,
  `sync/channel_shared.rs`, `sync/future.rs`, `sync/semaphore.rs`,
  `sync/mutex.rs`, `sync/join.rs`, `sync/waiters.rs`
- **Public:** `LocalChannel` and `SharedChannel` as two types rather than
  one with a flag — whether they share one API surface is open in
  `design/channels.md` and this row does not settle it — with their
  `Sender` and `Receiver`; `Future`, `Promise`, `Semaphore`, `Permit`,
  `Mutex`, `MutexGuard`, `Join`
- **Crosses its boundary:** the observables the detector reads —
  occupancy, remaining space, the closed flag with its reason, both end
  counts, a future's resolved and broken flags, a guard semaphore's holder
  list and free count, and the write end, read end and promise as distinct
  traced objects; the counted reference in a debtor's owner field, which
  it keeps truthful and leaves readable after the debtor terminates; the
  duty hook it installs for `sched`'s drain
- **Knows:** the two channel kinds and why they are two algorithms; the
  ends and the rule that no traced reference runs back from a resource to
  an end; closing and breaking; decide-and-publish under a leaf lock; the
  duty a supply wake carries and who re-runs it; whether a waiter is on
  this thread, which decides an inline wake against a forwarded one
- **Does not know:** how a coroutine suspends; how liveness is computed — the
  marks that do and do not cross the owner field are `deadlock`'s; what any
  other poster puts in an intake entry, its own carrying the resource
  beside the coroutine, the entry index, the epoch and the result
- **Depends on:** `coroutine`, `sched`, memory manager — a local channel lives
  in the thread's heap and a shared one in the immortal region, and the
  detector reads their kind from the class

### `deadlock` — proving a wait dead

- **Files:** `deadlock/mod.rs`, `deadlock/marks.rs`,
  `deadlock/liveness.rs`, `deadlock/attribution.rs`,
  `deadlock/watchdog.rs`, `deadlock/report.rs`, `deadlock/resolve.rs`,
  `deadlock/dump.rs`
- **Public:** `Detector`, `Mode`, `Report`, `ReportHandle`, `Dump`, the
  diagnostic channel
- **Crosses its boundary:** it rides the collector's walk and adds a
  second mark to it; it arms and removes a watchdog entry through the
  worker driver, never reaching the reactor directly; it posts a
  conditional resolution into a worker's intake queue and reads no answer
  back; it stamps the members of a published report; it installs the
  arming predicate `sched` evaluates at park and the condition a
  last-to-sleep worker tests; it reads the registered handle table off the
  collector's roots, which is why it names `io-api` nowhere
- **Knows:** the two marks and the three exceptions to the liveness mark's
  flow; the three resource kinds and the liveness rule of each; the
  fixpoint, its validation and the owner-side re-read; the attribution
  walks, which it owns outright; victim selection and the report stamp;
  **and the diagnostic dump**, which is not detection and is composed here
  because it is composed of `pool`'s walk and `sync`'s observables and
  published on the same channel — it answers what everything is waiting on,
  including the one case no fixpoint can report, a write end held by live
  code that never writes
- **Does not know:** how an operation is submitted, how a stack is
  allocated, how a channel is implemented — it reads the observables
  `sync` exposes and nothing behind them
- **Depends on:** memory manager, `coroutine`, `sync`, `sched`, `pool` (a
  slot's `kind` byte, and the dump), `switch` (the foreign-frame count)

### `reactor` — submitting operations and delivering completions

- **Files:** `reactor/mod.rs`, `reactor/driver.rs`, `reactor/submit.rs`,
  `reactor/complete.rs`, `reactor/multishot.rs`, `reactor/buffers.rs`,
  `reactor/wheel.rs`, `reactor/filepool.rs`, `reactor/uring.rs`,
  `reactor/iocp.rs`, `reactor/kqueue.rs`, `reactor/epoll.rs`
- **Public:** `Reactor`, `Backend`, `Submission`, `Completion`, `Timer`,
  the three buffer contracts, `submit`, `arm`, `disarm`
- **Crosses its boundary:** the entries it posts into `sched`'s intake
  queue, and the worker driver it implements, behind which its waker rings
  a sleeping owner; the timer wheel, which is the third memory root; the
  waiter cell it writes at arming and empties at completion or retire; the
  re-arm starvation report it publishes to the detector's diagnostic
  channel
- **Knows:** the four backends and what each owes; buffer ownership from
  submission to completion; multishot and the socket queue it appends to;
  the timer wheel; the thread pool that carries regular-file I/O on the
  readiness backends; which thread owns which ring
- **Does not know:** scheduling policy, liveness, what a channel is
- **Depends on:** `pool`, `sched`, `coroutine`, `deadlock` (the diagnostic
  channel it publishes into)

### `cancel` — ending work that is already in flight

Split across the two crates, and the halves know different things and
depend on different modules.

- **Files:** `cancel/core.rs` in `io-core`, `reactor/cancel.rs` in
  `io-reactor`
- **Public:** `cancel`, `CancelLevel`, `Answer`. `CancelHandle` is
  `coroutine`'s type; this module installs one and defines none
- **Crosses its boundary:** the promise a cross-thread request carries,
  resolved by the worker that applies it; the cancelled byte, which for a
  declared actor any thread may write; the cancel handle itself, the first
  of the three opaque callables
- **Knows:** core half — the cancelled bit, the claim of the winner field,
  the order of teardown, and what each of the four answers means.
  Operation half — what the kernel still owns, and the phases from
  emptying the waiter cell to the owed count reaching zero
- **Does not know:** core half — which backend owns the operation.
  Operation half — why the coroutine is being cancelled
- **Depends on:** core half — `coroutine`, `sched`, `switch`, `sync`, `pool`.
  Operation half — `reactor`, `pool`

### `io-api` — the public surface and the C ABI

- **Files:** `api/mod.rs`, `api/abi.rs`, `api/handles.rs`,
  `api/channel.rs`, `api/diag.rs`
- **Public:** the C ABI surface, and the Rust surface a consumer that owns
  its compiler uses instead
- **Crosses its boundary:** the registered handle table, which is a
  liveness root and holds the write ends of foreign holders; the receive
  contract, an output slot and three outcomes rather than two; the stack
  size hint; the diagnostic channel, polled from here; the notification
  that a coroutine was ended by a stack overflow
- **Knows:** what may cross the ABI and what may not; that no pool handle
  crosses it, which is what bounds the generation's staleness window; the
  registered handle table, which it implements — this is the one structure
  the crate owns rather than composes, and it owns it because a write end
  handed to a foreign holder is registered at the hand-out
- **Does not know:** how anything below it works; every other surface here
  is a composition
- **Depends on:** `io-core`, `io-reactor`

Three of its obligations are named in the corpus and designed nowhere: the
register and unregister calls of the handle table, how a foreign holder
dropping a write end becomes visible (`design/channels.md`), and the
death-notification path for a coroutine ended by a stack overflow
(`design/stacks.md`).

## Layering

**There is no total order over these modules, and an earlier draft of this
file that claimed one was wrong three times in a row.** The call graph is
acyclic, but it is not a stack of levels: the detector's arming rule runs
on the park path, the detector reads a table the API crate owns, and the
reactor publishes into the detector's channel. Every one of those looks
like an upward edge, and each is an upward edge of a different kind.

**Three kinds of edge, and only the first is a dependency.**

- **A call.** One module names another's type or function. These are what
  the table below records, and they are acyclic.
- **An installed callable.** The module that knows installs an opaque
  thing; the module that holds the boundary invokes it without looking
  behind it. Four exist: the cancel handle, the worker driver with its
  waker, `sync`'s duty hook, and the detector's arming predicate that
  `sched` evaluates at park. The invoker depends on the *contract*, never
  on the installer.
- **A read of a root.** The collector reaches a structure, and a module
  reads it there rather than through its owner. The registered handle
  table is such a root: `io-api` fills it, and `deadlock` reads it off the
  collector's walk without naming `io-api`.

The call graph:

| Module | Calls |
|---|---|
| `stack` | memory manager |
| `switch` | `stack` |
| `coroutine` | `stack`, `switch`, memory manager |
| `sched` | `coroutine`, `switch` |
| `pool` | `coroutine`, `sched`, memory manager |
| `sync` | `coroutine`, `sched`, memory manager |
| `deadlock` | `coroutine`, `sched`, `pool`, `sync`, `switch`, memory manager |
| `reactor` | `pool`, `sched`, `coroutine`, `deadlock` |
| `cancel`, core half | `coroutine`, `sched`, `switch`, `sync`, `pool` |
| `cancel`, operation half | `reactor`, `pool` |
| `io-api` | everything |

The one edge worth defending is `reactor` → `deadlock`: the reactor
publishes its re-arm starvation report into the detector's diagnostic
channel, and the detector reaches no reactor structure back, because the
watchdog is armed through the worker driver and the resolution goes into
`sched`'s intake queue.

`cancel` names more modules than anything else because it is the one
module allowed to tell `reactor` that an operation must end early, and the
only one that writes a coroutine's cancelled bit from outside `coroutine`.

## Shared resources

- **The object pools** are owned by `pool` and lent by reference.
  `reactor` allocates sockets, timers, operations and buffers from them; a
  caller that took a buffer under contract 1 and did not submit it returns
  it through the same public release. A waiter cell inside a slot belongs
  to the worker that owns the slot, and no other thread writes it.
- **Stacks** are owned by `stack` and lent to `coroutine`. A stack returns to
  its pool when its coroutine completes, which is sound only because nothing
  the kernel may touch after submission lives on a stack: buffers and
  submission structures come from the buffer pool instead.
- **Three structures are memory roots**, and between them they reach every
  coroutine that must not be collected. The **ownership table**, owned by
  `sched`, is a registry of lifetimes rather than a map of current owners:
  a coroutine joins its creating worker's shard when it is created and leaves on
  its terminal transition, so it reaches every registered coroutine and not only
  one whose wait is undecided. The **intake queues**, also `sched`'s, reach
  a coroutine whose wait is decided and whose cross-thread retire is still
  pending. The **timer wheel** is `reactor`'s. No pool is a root.
- **The actor's header** lives in the arena the actor owns and is not any
  module's to allocate. `sched` writes its readiness word and its queue
  link, any thread may push its mailbox, and `deadlock` reads all three.
  The arena is installed at mount time by `sched` and belongs to the
  consumer.
- **The collector's walk** is owned by the memory manager. `deadlock`
  extends it with a second mark and owns only the attribution walks that
  run inside the same pass.
- **The registered handle table** is `io-api`'s, and it is a liveness
  root. What registers and unregisters an entry is not designed.

## Hot paths

Named now so that measurement has a target later. Nothing here is
measured; `dev/BENCHMARKS.md` is created with the first figure.

| Path | Modules | What would be measured |
|---|---|---|
| the context switch | `switch` | cycles per switch, narrow and full |
| park and wake | `coroutine`, `sched` | a round trip on one thread, and across two |
| completion dispatch | `reactor` | per completion, and per batch drained |
| a channel send and receive | `sync` | the fast path with no waiter, and with one |
| enqueue into the ready set and ring a sleeper | `sched` | how it degrades as core count rises |

The last row is the one with a counterexample rather than a guess behind
it. Java's virtual-thread scheduler was profiled by Oracle on a 38-core
Xeon 8368 and lost 4.4x against platform threads on the same work at 32
cores, with 17% of all cycles in the single atomic its wake signal needs
(JDK-8360046, 2025-06-19). Neither the machine nor the workload is ours
and the figure transfers to nothing here; what transfers is where to point
the profiler first.
