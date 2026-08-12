# Deadlock Detection

## What is detected

A set of units is deadlocked when none of them can ever proceed, because
each waits on something only another member could supply. The substrate
detects that and reports it as data: which units, what each waits for, and
who each expected to supply it.

What it does not detect:

- **A wait that will end late is not a deadlock.** An operation in the
  kernel completes or errors; a timer fires.
- **A cycle through the outside world is invisible.** Two processes
  waiting on each other's sockets are deadlocked and the graph ends at the
  kernel.
- **An OR wait containing a timer is never proved deadlocked**, because
  the timer is a live half and any live half satisfies an OR. This does
  not extend to AND: a unit waiting for *both* a timer and a member of a
  dead set never proceeds, and is reported.

## Edges run unit → resource → unit

A wait record's entry does not name the unit that will end the wait. It
names a **resource** — a mutex, an actor, a channel, an operation, a timer
— and the resource's own slot records who currently owes it
(`design/pool.md`). Traversal is two hops, and the second is read fresh at
traversal time.

Recording a unit directly was the first design and it is wrong under
ownership transfer. Mutex M is held by unit 1; units 2 and 3 park on it,
both recording poster = unit 1. Unit 1 releases M, wakes unit 2, finishes,
and its slot is reused. Unit 3's record now names a slot holding a
stranger. Treating that as a dead edge invents a deadlock for unit 3 while
unit 2 is milliseconds from releasing M; treating it as live loses the
real case where a channel's last sender exits without sending. Neither
answer is right, so the edge does not go stale in the first place: unit 3
names M, and M names whoever holds it now.

Actors are resources too. A unit awaiting a reply names the actor; the
actor's slot carries its mailbox state and the handle of the unit
processing its current message, and traversal continues from there. That
is how an actor-mediated cycle closes, which is the case the actor model
raises as an open question and the reason this detector exists.

## The trigger: a watchdog on waits that could be cycles

A search runs when a wait has aged past a threshold. Two mechanisms were
possible and one of them does not exist: a parked unit has no worker and
no thread, so it cannot check its own age.

**A unit that parks arms a watchdog entry in the timer wheel** — the same
wheel the reactor already runs — and disarms it when the wait ends. The
wheel firing is what starts the search, on whichever worker services the
wheel.

**Only waits that could possibly be cycles arm one.** A wait with any half
that the kernel will end cannot be proved deadlocked, so it arms nothing.
That is what keeps the cost off the common path: a server with a hundred
thousand idle connections has a hundred thousand waits on the kernel and
zero watchdogs. Mutexes, channel receives and synchronous actor calls arm
one; reads, writes and accepts do not.

The threshold is PostgreSQL's arrangement in spirit — check after a wait
gets old rather than on every wait — but not its mechanism: there the
waiter is an operating-system process that can arm a real timer for
itself, and a parked unit is not.

## Blocked, and what the search reads

The search computes which units are permanently blocked: assume every
parked unit reachable from the suspect is blocked, then remove the ones
that are not, until nothing more can be removed.

**A half that has already fired is not a wait.** The search reads the
record's winner field, its `remaining` counter and each half's fired flag,
and considers only halves still outstanding. Ignoring them produces a
false deadlock in a system where nothing is moving at all: a unit in an
AND wait whose first half already fired, and whose second half is a kernel
operation, is waiting only on the kernel — but an algorithm that looks at
posters alone sees a half pointing back into the set and marks it blocked.

**A wait already decided is not a wait.** If the winner field is claimed,
the unit is owed a wake and only the state word has yet to move. Such a
unit is live no matter what its entries say.

A **half** is live when what it waits on will act without help from the
set: a kernel operation, an armed timer, a resource whose current owner is
runnable or outside the set.

A **unit** is not blocked when:

| Mode | Not blocked if |
|---|---|
| OR | **any** outstanding half is live |
| AND | **every** outstanding half is live |

Removing one unit can make another live, so removal repeats to a fixed
point. What is still marked and reachable from the suspect is deadlocked.

The two modes are why one algorithm covers both cases the literature
separates: under AND alone the answer is a cycle, under OR alone a knot,
and the modes are already in the records.

### A worked case

```
unit 1  AND  [ reply from actor B ]                     parked
unit 2  OR   [ mutex M, timer 5 s ]                     parked, running B's message
```

Unit 2 has a live half, so it is removed; unit 1 waits on actor B, whose
current unit is unit 2, which is not blocked, so unit 1 is removed. No
report, and rightly: in five seconds unit 2 resumes with a timeout.

Remove the timer and nothing is live. Unit 2 waits on M, held by unit 1;
unit 1 waits on B, whose current unit is unit 2. Both stay marked and the
pair is reported.

## Reading a moving system

The search reads slots other threads are writing. Four rules make its
answer sound, and each closes a specific way of being wrong.

- **Read only units whose state word says `Parked`,** re-read with
  acquire. A unit's epoch moves when it parks *again*, not when it wakes,
  so a woken-and-running unit is otherwise indistinguishable from an
  untouched one (`design/pool.md`).
- **Read the record as a seqlock.** The epoch is odd while entries are
  being written, and a reader that sees an odd or changed epoch discards
  the slot.
- **Read the decision fields, not only the entries** — winner,
  `remaining`, fired flags — because a waker changes those without
  touching the state word or the epoch. A search that ignores them reports
  a unit that is already owed a wake.
- **Re-validate the whole set before reporting.** Slots are validated at
  different instants, so a set assembled from them was never observed
  simultaneously. After the fixed point, every member's state word,
  generation, epoch and decision fields are re-read; any change abandons
  the search.

The result misses a real cycle occasionally and never invents one. A
missed cycle is found at the next watchdog expiry; a false one kills a
healthy unit.

### Two searches must not both act

Every member of a cycle, and every unit hanging off it, can expire
independently, so two workers can search overlapping sets at once, both
confirm, and both kill a victim.

**A search that reaches the confirmation stage claims its set**, with a
compare-and-swap on a claim word in the suspect's slot. This is the one
write the detector makes, and it is the exception to "the detector writes
nothing": a search that loses the claim abandons and lets the winner act.
The claim is released when the victim's cancellation is delivered.

## The victim

A confirmed set is reported and one member is failed so the rest proceed.

**The victim is chosen from the core** — the members that are actually in
the cycle or the knot — and never from a unit merely waiting on it. The
reachable set includes both, and failing a bystander leaves the cycle
intact and the next expiry kills another bystander.

Within the core, in order:

1. **Skip anything that cannot be failed.** A unit parked below a live
   foreign frame will never reach another suspension point, so cancelling
   it does nothing (`design/cancellation.md`). It is not chosen last; it
   is not chosen.
2. **Prefer a unit the consumer marked retryable.** The flag is set at
   creation and lives in the unit slot (`design/pool.md`); the consumer
   knows what is safe to repeat and the substrate does not.
3. **Prefer the youngest wait**, which has done the least work.

Failing a unit is cancellation, and cancellation is delivered through the
state word rather than through a half, which matters here: a wake aimed at
one half of an AND wait would decrement a counter and return without
waking (`design/cancellation.md`).

**When the core has no failable member**, the set is reported with no
victim. The process keeps running with that set stuck and the diagnostic
says so. The alternative is unwinding frames that were not compiled for
it, which trades a diagnosable hang for a crash of unknown shape.

## The report

A report is a value: the set, each member's identity and outstanding wait,
each resource and its owner, and which member was failed. It is built in a
report slot taken from a pool, and its handle travels two ways — to the
failed unit, as the result it resumes on, and to a diagnostic channel any
consumer can poll. The second path is what covers a set with no victim,
where nobody resumes to receive anything.

`php-src/ext/async` prints its wait graph to standard output under a debug
flag, which puts the information exactly where it cannot be used.

## What is shared with the collector

The collector walks object graphs and stamps its own epochs, so the walker
and the header discipline are worth sharing. **The epoch bytes are not
shared:** a collector stamp landing between a half being armed and its
completion would fail the waker's epoch check and lose the wake. Whether
anything beyond the walker is shared remains open in the decisions journal
and this document does not close it.

The trigger and the protocol are not shared, and the reason is sharp: a
deadlocked actor never reaches a message boundary, so it never answers the
collector's handshake. A deadlock stalls mark termination, and a detector
waiting on the collector would be unavailable exactly when it is needed
(`rfc/BACKLOG.md`, 2026-08-12). It needs no handshake, because it reads
only parked units.

## Cost

- **On the park path:** one watchdog insert, and only for a wait with no
  kernel half.
- **At expiry:** one traversal of the subgraph reachable from one unit,
  one confirmation pass, one claim.
- **Never:** a scan of the pool. That exists for the human-facing
  diagnostic dump and is not on this path (`design/pool.md`).

No figure is claimed. The traversal is bounded by reachability and how
large that is in practice is a measurement against real programs.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record, the epoch, the parked state | `design/execution.md` |
| slots, generations, resources and their owners | `design/pool.md` |
| delivering a failure to a unit | `design/cancellation.md` |
| what an operation waiting in the kernel means | `design/reactor.md` |

## Open questions

- **The threshold.** A deployment parameter with no measurement.
- **Correctness of consumer-supplied posters.** A mutex or channel built
  by the consumer over the C ABI must keep its resource slot's owner field
  truthful, and "never invents a deadlock" rests on that. The contract is
  named here and its enforcement is not designed.
- **Repeated searches on a stuck set with no failable member.** Every
  expiry re-detects it. A suppression window per set is the obvious
  answer; the claim word is probably where it lives.
- **Watchdogs on long healthy waits.** A mutex held legitimately for
  longer than the threshold arms a search that finds nothing, repeatedly.
  Whether the threshold alone is enough, or holders need to extend it, is
  not decided.
- **Report lifetime.** A report outlives the units it names, so the slots
  it points at may be reused before a consumer reads it. Copying the
  identities into the report is the obvious fix and costs a variable-size
  allocation the pools do not currently offer.
