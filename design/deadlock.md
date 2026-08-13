# Deadlock Detection

## What is detected

A coroutine is deadlocked when its wait cannot be ended by any continuation
of the program. The substrate finds every such coroutine in one pass, fails
one per stuck core with an exception, and reports the whole set as data:
which coroutines, what each waits on, and who could have supplied it.

Two shapes qualify and the second has no cycle at all:

- **A cycle.** Each member waits on a resource only another member can
  release: two coroutines on two mutexes, or a pair closed through an
  actor's mailbox.
- **A starved waiter.** A coroutine waits for a value from a channel or a
  future whose write end no live code can reach any more. Nobody owes it
  anything and nobody ever will.

What is not detected:

- **A wait that will end late is not a deadlock.** An operation in the
  kernel completes or errors; a timer fires.
- **A cycle through the outside world is invisible.** Two processes waiting
  on each other's sockets are deadlocked and the graph ends at the kernel.
- **A write end held by live code that never writes.** Reachability proves
  that a value can still be produced, never that it will be. A receiver
  starved by a reachable, silent writer waits forever and is not reported;
  the diagnostic dump over the pools is the tool for that case.

## Terms

A wait record holds one **entry** per thing a coroutine waits on: the
resource, named by its handle when it is pooled and by a counted reference
otherwise (`design/execution.md`), a cancel handle, a result slot, a fired
bit. Beside the
entries the record carries the mode, the wait epoch, the winner field and
`remaining` (`design/execution.md`). The kind is a property of the resource
and is read from the resource itself — the class of a heap entity, the slot
header of a pooled one — because an entry copying it would let the copy
disagree with the object.

A wait **edge** has two ends and neither is a record field. The coroutine's
end names a resource. The resource's end answers who can end the wait, and
it is read fresh at evaluation, because ownership moves: two coroutines
park on mutex M and both record its holder, the holder releases M and
finishes, and the second record still names a coroutine that owns nothing
and will end no wait.

## Resources split three ways

The kind decides the liveness rule and nothing else about a resource does.

- **External** — a kernel operation, an armed timer, and the completion
  queue of a socket a multishot operation appends to
  (`design/reactor.md`). The kernel or the wheel serves it without any
  coroutine.
- **Debtor** — a mutex, a synchronous actor call, a join on a coroutine,
  and a semaphore whose permits are released only by dropping a permit
  guard. The resource names who owes it: the holder, the callee, the
  target, or for the semaphore its permit holders and its free count. **That
  field holds a counted reference** — counted so that a terminated debtor is
  still there to be read, since the rows below distinguish one from a
  running holder; traced by M like every counted reference here; crossed by
  neither L nor the attribution walk. This document owns the sentence,
  because it is the only reader of the field and the debtor resources have
  no document of their own; a synchronisation-primitives document, when one
  exists, inherits the execution half as `design/channels.md` carries the
  supply half today.
- **Supply** — a channel, a future, an actor mailbox, and any semaphore
  that can be posted without being acquired first. No owner field can
  exist, because whoever holds the write end may serve the wait. A future
  is a memory cell someone may write, and asking it who owes the value has
  no answer.

The semaphore falls on the line, so where it falls is decided by its API
rather than by its name; `design/channels.md` fixes that API and gives the
substrate's own semaphore no bare post. A permit released only by dropping a
guard ties release to acquisition, which is what makes a holder a debtor. A semaphore
with a post that anyone may call — `sem_post`, `ReleaseSemaphore`, Java's
`release`, and every semaphore reached over the C ABI, whose discipline we
cannot see — has no such tie: a semaphore created with zero permits and
posted by a producer that never acquired anything has no holders at all,
and classifying it as a debtor would report its waiter as deadlocked while
the producer is running.

A supply resource imposes one constraint on its own object model, and
`design/channels.md` carries it as a rule for the implementation: its ends
are reachable from their holders and the resource is reachable from its
ends, and there is no traced reference the other way. A count of live ends
is fine; a reference is not.

**Its queues of waiters are counted and traced, and two walks skip them**, and
this half is as load-bearing as the first. The link has to be counted, because
whoever arms an entry holds a reference so that a unit cannot be freed under a
wake still on its way, and it has to be traced, because a counted reference the
tracer cannot enumerate is unsound in this memory model. What it must not do is
conduct L: the flow rule below stops L leaving the cells of a parked coroutine
and does not stop it entering the coroutine object, so one live holder of a read
end would otherwise mark every coroutine parked on that resource and none could
enter the dead set. The attribution walk skips the same links, or a dead
coroutine holding a read end reaches other coroutines' write ends through the
queue, the dead graph gains edges that do not exist, and a victim is chosen from
a distorted set of sinks. `design/channels.md` states what an implementer checks
at each enqueue and unlink. With a back-reference, any live holder of the
read end makes the write end reachable through the resource, every channel
with a live receiver reads as servable, and supply resources stop being
detectable at all.

## The trigger

A trigger starts a pass. It never proves anything, and the fixpoint below
is the only thing that declares a deadlock.

**A coroutine that parks arms a watchdog in the timer wheel** — the wheel
the reactor already runs — and disarms it when the wait ends. Expiry
requests a collector epoch. A parked coroutine has no thread and cannot
check its own age, which is why the wheel holds the entry instead.

**Only a wait that could be proved dead arms one.** Call an entry
cycle-capable when it names a mutex, a semaphore, a channel, an actor, a
future or a join; a kernel operation, a timer and a socket's completion
queue are not. Then:

- an **OR** wait arms a watchdog when **every** entry is cycle-capable,
  because one kernel or timer entry satisfies an OR forever;
- an **AND** wait arms one when **at least one** entry is cycle-capable,
  whatever else it names — a kernel answer does not release a mutex;
- a **single-entry** wait follows the OR rule, the two modes agreeing
  there.

Both directions are needed. Without the AND rule, B waiting on [kernel
operation, mutex N] arms nothing and the cycle through N is found only at
quiescence. Without the OR rule for all-cycle-capable waits, three
coroutines each parked in OR over the other two's mutexes form a knot in
which nothing arms anything.

Cost on the common path is unchanged: a socket read is a single kernel
wait or an OR wait with a timeout, so a server with a hundred thousand
idle connections arms nothing.

**The last worker about to sleep requests a pass too**, when no operation
is in flight, no timer is armed and the shared ready set is empty. The last
worker is the one whose idle bit completes the mask, and it reads the ready
set's emptiness under the mutex it is about to release; no other worker's
private lists are read, because a worker holding anything in its lists has
not published its bit (`dev/DECISIONS.md`, 2026-08-13). This detects a
total freeze at the moment it forms rather than a threshold later. The
condition may be read racily, because a needless pass is only a pass. Go's
`checkdead` is this trigger and only this one, which is why a cycle of
three goroutines inside a running program is never found there.

Threshold: 1 s, in PostgreSQL's spirit, doubled on each re-arm of the same
wait epoch, so a mutex held legitimately for minutes costs a thinning
series of passes. Not measured; a deployment parameter.

## Where the pass runs

Inside the collector's walk phase, on the collector's thread. Two reasons,
and they point the same way. The collector already reads other threads'
entities safely, through block snapshots and a re-read of recorded cells in
a later phase, and the detector needs exactly that and nothing more. And a
deadlocked actor never reaches a message boundary, so it never answers the
mark-termination handshake: a detector placed after termination would be
blocked by the condition it exists to find.

## The algorithm

The wait-for graph is never built. Liveness is computed over the object
graph the collector walks anyway, and the dead set falls out as what the
computation did not reach.

### Two marks

One walk propagates two bits that differ in roots and in flow.

**M, the memory mark.** Unchanged: every root the collector already has,
including the scheduler's ownership table, the timer wheel and the reactor
intake queues, flowing through everything. **No pool is a root.** The
reference in an armed waiter cell names a unit the ownership table still
registers, because a unit leaves that table only on its terminal transition
and reaches it only by resuming, which requires the wait decided; and once
the wait is decided the only cell emptied across threads is a socket's,
whose pending retire is an intake entry carrying its own counted reference
to the unit (`design/pool.md`, `design/reactor.md`).

**L, the liveness mark.** Roots: globals, the C-ABI registered handle
table, every coroutine that is neither parked nor `Terminal`, every parked
coroutine whose wait is already decided (winner claimed or `remaining`
zero), and every coroutine named by a pending intake entry whose
epoch matches.

A `Terminal` coroutine is excluded because it has finished and still
exists: whoever armed an entry against it holds a counted reference
(`design/execution.md`), so the object outlives its own completion. Rooting
it would contradict the three rows below that call a terminated holder
dead — a mutex it never released would mark its waiter live for as long as
anything referenced the corpse.

L flows through every object except three. It does not flow out of the cells
of a parked coroutine that is not itself L-marked; it does not flow through
the waiter-queue links of a supply resource; and it does not flow through
the fields by which a debtor names its debtor — a mutex's holder, a
guard-semaphore's holder list, an actor's current unit, a join's target. The
attribution walk crosses neither of the last two kinds of link. All of them
are counted references and the memory mark traces them like any other,
because a counted reference the tracer cannot enumerate is unsound here — an
untraced queue could never have carried a counted one.

**A debtor row reads the named coroutine's own mark**, and a field that
conducted L into it would satisfy the row from the row's own premise: a
mutex held in any global would mark its holder live, its waiter would read
"the holder is L-marked", and no cycle through that mutex could ever be
reported. Nothing legitimately reads a debtor resource's own liveness, so
cutting the flow costs nothing.

**Beside the marks the pass collects the served set**, which is not a mark
and does not propagate: a resource is *served* when a pending intake entry
would move it so a wait on it can proceed. The forms this takes are
examples and not a closed list — a deposit for a channel receive or a
future await, a take for a channel send, a post for a semaphore acquire —
and the rule is the property, because a resource kind added later brings
its own form. Every one of them is recorded nowhere else: a cross-thread
send whose sender has already finished leaves only the queue entry, and so
do a take that would free space in a full channel and a post whose poster
has terminated. Without the served set the resource reads as unservable and
its waiter is reported dead. The set is read by
the liveness table below and by nothing else, which is why it is a set of
resources rather than a mark: L-marking a channel object would say nothing
about whether anyone can still write to it.

The intake queue therefore has to be one queue per worker, drained in
order. Resolutions and deposits are posted to the same place, so a deposit
recorded during the pass is delivered before the resolution that the pass
would have based on its absence; two queues, or a queue drained out of
order, would let a healthy waiter be failed.

Rooting L in the scheduler's table would L-mark every parked coroutine and
report a deadlock never again. The memory roots are the wrong roots for
liveness, which is why the second bit exists.

### Liveness per entry

A fired entry is not outstanding and is not evaluated. Ignoring that is how
a set waiting only on the kernel gets reported: an AND wait whose first
entry has fired and whose second is a kernel operation waits on the kernel
alone, while an algorithm reading owners alone sees an entry pointing back
into the set.

An outstanding entry is live when:

| Resource | Live when |
|---|---|
| kernel operation | always: the kernel owns it |
| timer | always: it is armed |
| socket completion queue, multishot | always: the kernel appends while the series is armed, and between a series' end and the re-arm the reactor owes the re-arm (`design/reactor.md`) |
| mutex | the holder is L-marked. **Dead, and no edge**, when the holder has terminated: a finished coroutine never releases |
| semaphore, guard-released | the free permits plus the permits held by L-marked holders cover what the waiter asked for. **Dead, and no edge**, for a permit held by a terminated coroutine: teardown drops its guards and returns the permit, so a terminated holder still counted is a leak. One edge per dead holder of a needed permit |
| semaphore, postable | the free permits cover the request, or the semaphore is served, or the semaphore object is L-marked — over the C ABI, its entry in the registered handle table. Holders are not read: whoever reaches it can post |
| join | the target is L-marked or terminal, a terminal target's completion wake being in flight |
| actor call | the callee's current unit is L-marked, or the callee's readiness word reads ready or running, or its mailbox is non-empty. The readiness word is load-bearing rather than a convenience: between the worker taking a message out of the mailbox and creating the unit for it, the other two clauses are both false, and without this one a healthy synchronous call is reported deadlocked (`dev/DECISIONS.md`, 2026-08-13) |
| channel receive | the buffer is non-empty, or the channel is served, or the channel is closed, or the write end is L-marked |
| channel send | the buffer has space, or the channel is served by a pending take, or the channel is closed, or the read end is L-marked |
| future await | it is resolved or broken, or the future is served, or the promise is L-marked |

**The socket queue's row could not read otherwise.** Calling it dead while
the series is un-armed would require proving that no buffer ever frees,
which a pass cannot know and which is false in the routine `-ENOBUFS` gap
`design/reactor.md` describes: a pass landing in that gap would invent a
deadlock against a loaded, healthy connection. The one state this row
cannot report — a waiter over a queue the reactor can never re-arm — is a
fact the reactor already holds, and the reactor publishes it.

**Closed** is read as the closed flag or the relevant end count at zero —
write ends for a receive or an await, read ends for a send
(`design/channels.md`). The decrement that reaches zero is the moment the
resource becomes closed, so the window between it and the flag store is read
as closed rather than as a resource with no reachable write end, which is
what would otherwise report an invented deadlock against an ordinary
producer's exit.

A coroutine in **OR** mode is live when any outstanding entry is live; in
**AND** mode when every outstanding entry is live.

Liveness is per entry and never per coroutine. Marking a whole coroutine
live because one entry waits on the kernel loses a real deadlock for good:
B waits on AND [kernel operation, mutex N held by A], A waits on mutex M
held by B, the kernel answers, and B still never proceeds because N comes
only from A and A waits on B.

### The fixpoint

Seed L with its roots, then grow. When a parked coroutine turns live it is
L-marked and its cells are traced for L, because objects reachable only
through it may be the write ends other entries depend on. Each suspect
resource keeps a waiter list for the pass, so a write end gaining L or a
debtor turning live re-evaluates exactly the waiters that could change.

L only grows and a coroutine crosses dead-to-live at most once, so the work
list drains and the pass terminates.

Growing from the live set is the direction that works. Started from "every
coroutine is live" and pruned, a cycle counts itself as its own support and
survives every round.

The two modes are why one computation covers both cases the literature
separates: under AND alone deadlock is a cycle, under OR alone it is a
knot, and the modes are already in the records.

### The dead graph and its sinks

The dead set is every parked, undecided coroutine left without L. On it a
small explicit graph is built, one edge per outstanding entry:

- a **debtor** entry points at its debtor, which is necessarily in the dead
  set, because a live debtor would have made the entry live;
- a **supply** entry points at every dead coroutine from whose cells the
  write end is M-reachable, found by one attribution walk per dead
  coroutine over M-marked objects;
- a supply entry whose write end is M-reachable from nothing at all gets no
  edge. The fact is **grounded**: nobody can write, and that write end is
  garbage in this very pass.

**The cores are the sink strongly-connected components.** A cycle is a sink
of two or more, a starved waiter is a sink of one, and bystanders hang
above the sinks and are never chosen — failing a bystander leaves the cycle
intact and the next pass fails another bystander.

One coroutine per sink is resolved per pass. That is the largest sound
batch: resolving one invalidates every deadness proof that depended on it,
and a sink's proofs depend only on its own members and on grounded facts.

Within a sink, in order:

1. **Skip anything that cannot be failed.** A coroutine parked below a live
   foreign frame never reaches another suspension point, so the exception
   has nowhere to surface (`design/cancellation.md`). It is not chosen
   last; it is not chosen.
2. **Prefer a coroutine the consumer marked retryable.** The consumer knows
   what is safe to repeat and the substrate does not.
3. **Prefer the youngest wait**, which has done the least work.
4. **Then the lowest id**, so the choice is reproducible.

### A worked case

```
unit 1  AND  [ reply from actor B ]                     parked
unit 2  OR   [ mutex M, timer 5 s ]                     parked, running B's message
```

Unit 2 has a live entry, so it is L-marked; unit 1 waits on actor B, whose
current unit is unit 2, now live, so unit 1 is L-marked too. Nothing is
reported, and rightly: in five seconds unit 2 resumes with a timeout.

Remove the timer and neither is seeded. Unit 2 waits on M held by unit 1;
unit 1 waits on B, whose current unit is unit 2. Both stay unmarked, they
form one sink, and one of them is failed.

Chain the starved case onto it. Unit 3 parks receiving on channel D whose
write end is reachable from nothing; unit 3's frames hold the promise of
future F; unit 4 awaits F. Both are dead, the graph has one edge, unit 4 to
unit 3, so unit 3 is the sink and only unit 3 is failed. Unit 3 resumes
with the closed-channel error, and when its frames drop the promise, F
breaks and unit 4 fails with it — in the wake of the same pass.

## Reading a moving system

Slots are read while other threads write them, so a set assembled from them
was never observed at one instant. Two mechanisms make the answer sound,
and only the second is authoritative.

**The collector re-reads before reporting.** In the collector's later
re-read phase, per dead member: the state word still says `Parked`, the
wait epoch is unchanged, the winner is unclaimed, `remaining` is unchanged,
the fired bits are unchanged. Any difference drops that member's whole
weakly-connected dead component, whose deadness was computed from that
record. The epoch alone is not enough, because a waker changes fired bits,
`remaining` and the winner within one epoch while the state word stays
`Parked`.

**The owner thread confirms before acting.** The collector writes nothing
into a coroutine. It posts a conditional resolution into the owner's intake queue,
and the owner acts only if state, epoch, winner and fired bits still equal
what was recorded. Every input to deadness except the fired bits and the
winner is written once per epoch, and those two only grow within one, so
equality proves the snapshot described this exact wait however stale the
cross-thread reads were.

The worker intake queues are scanned once per pass, before the fixpoint. A
wake posted by a coroutine that has since died leaves the queue entry as
its only trace, and a pass that misses it fails a coroutine that is owed a
wake.

The result misses a real deadlock occasionally and never invents one. A
missed one is found at the next expiry; a false one raises an exception in
healthy code, and no catch block repairs a receiver that observed a closed
channel while another goes on reading from it.

## The protocol

Threads: **K** the collector, **O** an owner thread, **W** any worker.

**Trigger.**

1. **(W, at park)** If the wait can be proved dead by the arming rule,
   insert a watchdog entry; its deadline doubles on each re-arm of the same
   epoch. No ordering against the `Parked` store is required, because
   expiry only requests a pass.
2. **(W, at wait end)** Remove the entry. A stale expiry racing the removal
   requests a pass that finds nothing.
3. **(W, wheel service or last-to-sleep)** Set the collector's detection
   request and start an epoch if none is queued.
4. **(K)** Clear the request at the start of the pass, never at the end.
   Cleared at the end, a request that arrived mid-pass is lost and its wait
   waits for an unrelated pass.

**Pass**, in the walk phase.

5. **(K)** At each walked coroutine whose snapshot state is `Parked`,
   record epoch, mode, fired bits, winner, `remaining`, every entry, and
   the foreign-frame count. Recorded in the same visit that carries the M
   mark: a second visit later would mix two states of one record.
6. **(K)** Scan every worker intake queue: seed L with the coroutines its
   entries name, and the served set with the resources those entries would
   move — deposits and takes alike. Must precede the fixpoint, for the
   reason above.
7. **(K)** Seed the remaining L roots and run the fixpoint. Empty dead set
   ends the detector's part; the memory work continues either way.
8. **(K, re-read phase)** Validate every dead member, and for a supply entry
   validate the resource fields the verdict consumed as well: occupancy, the
   closed and broken flags, both end counts, a future's resolved state. Any
   movement drops the whole weakly-connected component, exactly as a movement
   of a record does. Must sit behind the collector's phase fence: a re-read
   servable from the same instant as the first read validates nothing. Without
   the resource half, a write-end count reaching zero inside the pass survives
   validation and the report published at step 10 is a lie even under
   `report-only`.
9. **(K)** Run the attribution walks, build the dead graph, compute the
   sinks, choose one target per sink, skip sets whose members all carry a
   current report stamp. Must follow 8, or a target is chosen from a set
   that validation would have dropped.
10. **(K)** Serialize the report, publish its handle to the diagnostic
    channel, stamp the members. Must precede 11: the resumed coroutine
    carries the report handle, and resolved first it would resume pointing
    at a report that does not exist yet.
11. **(K)** Post into each target's owner's intake queue a conditional resolution
    carrying the coroutine reference, the epoch, the fired bits, the error
    value and the report handle.
12. **(O, intake drain)** If state, epoch, winner and fired bits match — and,
    for a supply entry, if the resource's own fields still read as they did:
    store the error and the report handle in the record's own result slot,
    claim the winner with the reserved no-entry value (`design/execution.md`),
    retire every outstanding entry through its cancel handle, store `Woken`,
    enqueue.
    The owner-side re-read costs one read per resolution, and resolutions are
    rare; it is what makes a deposit that landed during the pass authoritative
    over a verdict built on its absence.
    That is the wake protocol's own order, and the state store is last for
    its reason (`design/execution.md`): a coroutine enqueued before its
    result is stored can be picked up by a worker and read a result that
    is not there. Otherwise drop the resolution: the record moved, the
    finding is stale, and a real set is found again next pass.

## Resolution is an exception, not a kill

The wait is decided with an error and the coroutine resumes at its wait
point, keeping its stack, its arena and its future work. The process keeps
running.

**What is raised.** A channel receive or send fails exactly as it fails on
a closed channel; a future await fails exactly as it fails on a broken
promise. Code already correct against closure is correct against deadlock
with no change, and the difference is carried in the error value: a cause
naming deadlock, and the report handle. A mutex, an actor call and a join
have no closure to imitate and raise a deadlock error of their own.

**The detector marks no resource.** Resources die by their own rule, and
`design/channels.md` states it: dropping the last write end closes a channel
and breaks a future, waking every waiter through the ordinary path. That covers the co-waiters without
a second pass, because the resumed coroutine drops the last write end while
unwinding. A mark placed by the detector would be wrong in the one case
that matters — a write end still held by members of the dead set, where
resolving one may resurrect a legitimate writer, and the mark would turn
its legitimate write into an error the detector invented.

**Other waiters on the same resource** are served by consequence: a
released mutex wakes the next waiter normally, a dropped write end fails
every co-waiter at once. What consequence does not reach is found at the
next expiry.

**Where this does not apply.** A sink whose every member carries a live
foreign frame cannot be resolved, because the exception has no suspension
point to surface at. The set is reported with no target and suppressed
until a member's epoch moves. Unwinding frames we did not compile trades a
diagnosable hang for a crash of unknown shape.

## The report and the policy

A report is a raw serialized buffer, not an entity: heaps are per thread
and the collector may not create objects in another thread's heap, and
copied identities keep the report readable after the coroutines it names
are gone. It carries the pass id; per member the id, class, mode, epoch and
per outstanding entry the resource kind, its id and its debtor or write-end
holders at snapshot; the sinks; and each target with its disposition.

Its handle travels two ways: to the failed coroutine, as part of the error
it resumes on, and to a diagnostic channel any consumer can poll. The
second path is what covers a set with no target, where nobody resumes to
receive anything. The channel is a bounded ring with a dropped count. A
per-process counter of resolutions is what distinguishes a quiet system
from one that swallows deadlocks, which is the cost of resolving them
softly.

`php-src/ext/async` prints its wait graph to standard output under a debug
flag, which puts the information exactly where it cannot be used.

**Policy is the consumer's, on the scheduler, process-wide:**

| Policy | Effect |
|---|---|
| soft (default) | publish the report, resolve one coroutine per sink |
| hard | publish the report, abort the process — for tests and CI, where a deadlock must be loud |
| report-only | publish and stamp, resolve nothing — for observation in production |

Per-wait overrides are not offered; the retryable flag is the per-coroutine
knob and it selects the victim rather than the policy.

## Cost

- **On the park path:** one watchdog insert and one removal, only for a
  wait the arming rule admits.
- **Per pass, over the walk that runs anyway:** one class test per visited
  object; roughly two cache lines copied per parked coroutine, plus one
  pointer chase when a record has more than two entries; one extra bit
  tested and set per traversed edge; at most one re-trace per coroutine
  that turns live; an intake scan proportional to pending entries.
- **Per trigger:** one collector epoch that would not otherwise have run.
  This is the real cost of the design, bounded by the watchdog's doubling.
- **Never:** a scan of the pools, by either mark. The walk of slot headers
  exists for the human-facing diagnostic dump and is on neither path
  (`design/pool.md`).

No figure is claimed. Latency is the threshold plus the walk for a partial
deadlock, and the walk alone for a total freeze.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record, the epoch, the parked state, the wake order | `design/execution.md` |
| which waitable resources are pool slots, and where the rest live | `design/pool.md` |
| delivering a failure to a coroutine, and the foreign-frame count | `design/cancellation.md` |
| what an operation waiting in the kernel means | `design/reactor.md` |
| why the detector runs inside the collector | `dev/DECISIONS.md`, 2026-08-12 |

## Open questions

- **Integration with the `ll-model` walk is asserted, not verified.** The
  second mark with its own flow rule, the re-visit of a coroutine that
  turns live mid-fixpoint, and multi-word snapshots read across threads —
  of a wait record, and of a resource's own fields, a mutex holder or a
  channel's occupancy and closed flag — under a re-read discipline built
  for single counted cells: none of the three is checked against
  `run_epoch` and the steppable collector. The fallback is a detector-owned
  second traversal of parked coroutines inside the same pass, which changes
  this document's cost section and not its protocol. **A fourth item joins
  them**: a counted reference hands off between two roots mid-pass, from an
  armed waiter cell to the intake entry of a cross-thread retire
  (`design/pool.md`). Whether the barrier discipline that covers any root
  mutation covers this one is the same unverified question, and if
  `ll-model` requires every counted in-edge enumerated per object, the
  cell's reference is enumerated at the unit's own record — one in-edge per
  outstanding entry naming a pooled resource.
- **The intake queue's ordering contract is owed by the scheduler**
  (`dev/DECISIONS.md`, 2026-08-13), and no document describes the queue's
  order yet. Step 12 and the served
  set both rest on one queue per worker drained in order, so a priority
  lane for cancels — an optimization that document's own motivation invites
  — would fail a healthy waiter.
  A total order across producers is the costly half of that contract, and a
  deposit and a resolution do come from different producers. The structure
  that gives such an order is a bounded ring with one shared index, and
  TrueAsync's server measured rings of that shape at 2.3 M ops/s against
  56–68 M for per-producer sub-queues at eight producers
  (`true-async-server/deps/concurrentqueue/UPSTREAM.md`: i7-11700K, pointer
  payload, capacity 4096, that project's workload rather than ours). Three
  answers are open: pay for the shared index; carry resolutions on a
  separate lane whose only producer is the collector, drained after the
  intake queue is empty; or have the owner re-read the resource before
  applying a resolution, which costs one read per resolution and makes the
  order irrelevant.
- **A write end the tracer cannot reach is indistinguishable from one that no
  longer exists**, and this is a hole rather than a caveat. Attribution walks
  the cells of a coroutine, which the `walk` hook makes reachable; a write end
  in a *frame* of a stackful coroutine is in no cell, no document here designs
  enumerating references out of a coroutine's stack, and for a frame someone
  else compiled it cannot be done. A producer that computes for a while with
  the write end in a local variable therefore reads as unreachable, and its
  healthy receiver is failed with an invented deadlock. Two closures exist and
  neither is designed: stack maps from `ll-model`, or registration of such an
  end as a C-ABI holder's is registered. Until one lands, a resource whose
  write ends are not all traceable or registered has to be treated as always
  live, which costs detection for the most ordinary producer there is.
- **A wait on an actor's mailbox has no row.** The kinds above list the mailbox
  as a supply resource, and the table has a row for a synchronous actor call,
  which is a debtor. A coroutine waiting for a message of its own — an actor
  reading its mailbox — is neither, and nothing here says when that wait is
  live. The observables exist: the readiness word and the queued count
  (`dev/DECISIONS.md`, 2026-08-13). The row does not.
- **The threshold.** A deployment parameter with no measurement.
- **Correctness of consumer-supplied resources.** A mutex or channel built
  over the C ABI must keep its owner field truthful and its write ends in
  the registered handle table, and "never invents a deadlock" rests on
  that. Such a resource is treated as always live until it adopts the
  contract, so the default costs detection rather than correctness. The
  enforcement is not designed.
