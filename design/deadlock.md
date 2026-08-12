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
resource handle, a cancel handle, a result slot, a fired bit. Beside the
entries the record carries the mode, the wait epoch, the winner field and
`remaining` (`design/execution.md`). The kind is a property of the resource
and is read from the resource itself — the class of a heap entity, the slot
header of a pooled one — because an entry copying it would let the copy
disagree with the object.

A wait **edge** has two ends and neither is a record field. The coroutine's
end names a resource. The resource's end answers who can end the wait, and
it is read fresh at evaluation, because ownership moves: two coroutines
park on mutex M and both record its holder, the holder releases M and
finishes, its slot is reused, and the second record now names a stranger.

## Resources split three ways

The kind decides the liveness rule and nothing else about a resource does.

- **External** — a kernel operation, an armed timer. The kernel or the
  wheel serves it without any coroutine.
- **Debtor** — a mutex, a synchronous actor call, a join on a coroutine,
  and a semaphore whose permits are released only by dropping a permit
  guard. The resource names who owes it: the holder, the callee, the
  target, or for the semaphore its permit holders and its free count.
- **Supply** — a channel, a future, an actor mailbox, and any semaphore
  that can be posted without being acquired first. No owner field can
  exist, because whoever holds the write end may serve the wait. A future
  is a memory cell someone may write, and asking it who owes the value has
  no answer.

The semaphore falls on the line, so where it falls is decided by its API
rather than by its name. A permit released only by dropping a guard ties
release to acquisition, which is what makes a holder a debtor. A semaphore
with a post that anyone may call — `sem_post`, `ReleaseSemaphore`, Java's
`release`, and every semaphore reached over the C ABI, whose discipline we
cannot see — has no such tie: a semaphore created with zero permits and
posted by a producer that never acquired anything has no holders at all,
and classifying it as a debtor would report its waiter as deadlocked while
the producer is running.

A supply resource imposes one constraint on its own object model: its ends
are reachable from their holders and the resource is reachable from its
ends, and there is no traced reference the other way. A count of live ends
is fine; a reference is not. With a back-reference, any live holder of the
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
future or a join; a kernel operation and a timer are not. Then:

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
is in flight, no timer is armed and every run queue is empty. This detects
a total freeze at the moment it forms rather than a threshold later. The
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
intake queues, flowing through everything.

**L, the liveness mark.** Roots: globals, the C-ABI registered handle
table, every coroutine that is not parked, every parked coroutine whose
wait is already decided (winner claimed or `remaining` zero), and every
coroutine named by a pending reactor-intake entry whose epoch matches.

L flows through every object except that it does not flow out of the cells
of a parked coroutine that is not itself L-marked.

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
| mutex | the holder is L-marked. **Dead, and no edge**, when the holder has terminated: a finished coroutine never releases |
| semaphore, guard-released | the free permits plus the permits held by L-marked holders cover what the waiter asked for. **Dead, and no edge**, for a permit held by a terminated coroutine: teardown drops its guards and returns the permit, so a terminated holder still counted is a leak. One edge per dead holder of a needed permit |
| semaphore, postable | the free permits cover the request, or the semaphore is served, or its handle is L-marked. Holders are not read: whoever reaches the handle can post |
| join | the target is L-marked or terminal, a terminal target's completion wake being in flight |
| actor call | the callee's current unit is L-marked, or the callee has a queued unit or a non-empty mailbox |
| channel receive | the buffer is non-empty, or the channel is served, or the channel is closed, or the write end is L-marked |
| channel send | the buffer has space, or the channel is served by a pending take, or the channel is closed, or the read end is L-marked |
| future await | it is resolved or broken, or the future is served, or the promise is L-marked |

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
into a coroutine. It posts a conditional resolution to the owner's reactor,
and the owner acts only if state, epoch, winner and fired bits still equal
what was recorded. Every input to deadness except the fired bits and the
winner is written once per epoch, and those two only grow within one, so
equality proves the snapshot described this exact wait however stale the
cross-thread reads were.

The reactor intake queues are scanned once per pass, before the fixpoint. A
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
6. **(K)** Scan every reactor intake queue: seed L with the coroutines its
   entries name, and the served set with the resources those entries would
   move — deposits and takes alike. Must precede the fixpoint, for the
   reason above.
7. **(K)** Seed the remaining L roots and run the fixpoint. Empty dead set
   ends the detector's part; the memory work continues either way.
8. **(K, re-read phase)** Validate every dead member. Must sit behind the
   collector's phase fence: a re-read servable from the same instant as the
   first read validates nothing.
9. **(K)** Run the attribution walks, build the dead graph, compute the
   sinks, choose one target per sink, skip sets whose members all carry a
   current report stamp. Must follow 8, or a target is chosen from a set
   that validation would have dropped.
10. **(K)** Serialize the report, publish its handle to the diagnostic
    channel, stamp the members. Must precede 11: the resumed coroutine
    carries the report handle, and resolved first it would resume pointing
    at a report that does not exist yet.
11. **(K)** Post to each target's owner reactor a conditional resolution
    carrying the coroutine reference, the epoch, the fired bits, the error
    value and the report handle.
12. **(O, reactor drain)** If state, epoch, winner and fired bits match:
    store the error as the result, claim the wait as decided, retire every
    outstanding entry through its cancel handle, store `Woken`, enqueue.
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

**The detector marks no resource.** Resources die by their own rule:
dropping the last write end closes a channel and breaks a future, waking
every waiter through the ordinary path. That covers the co-waiters without
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
- **Never:** a scan of the pools. That exists for the human-facing
  diagnostic dump and is not on this path (`design/pool.md`).

No figure is claimed. Latency is the threshold plus the walk for a partial
deadlock, and the walk alone for a total freeze.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record, the epoch, the parked state, the wake order | `design/execution.md` |
| slots, resources and their owners | `design/pool.md` |
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
  this document's cost section and not its protocol.
- **Closing on the drop of the last write end** is channel and future
  semantics, and this document depends on it while no document defines it.
  The object-model constraint above belongs there too, and so does the
  substrate semaphore's release API: this document's debtor branch holds
  only if a permit is released by dropping a guard, which constrains an
  API that is not designed yet.
- **The intake queue's ordering contract is owed by `design/reactor.md`**,
  which today describes the queue as carrying cross-thread cancels and
  migrated submissions and says nothing about order. Step 12 and the served
  set both rest on one queue per worker drained in order, so a priority
  lane for cancels — an optimization that document's own motivation invites
  — would fail a healthy waiter.
- **The threshold.** A deployment parameter with no measurement.
- **Correctness of consumer-supplied resources.** A mutex or channel built
  over the C ABI must keep its owner field truthful and its write ends in
  the registered handle table, and "never invents a deadlock" rests on
  that. Such a resource is treated as always live until it adopts the
  contract, so the default costs detection rather than correctness. The
  enforcement is not designed.
