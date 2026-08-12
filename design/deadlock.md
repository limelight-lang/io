# Deadlock Detection

## What is detected

A set of units is deadlocked when none of them can ever proceed, because
each is waiting on something only another member of the set could supply.
The substrate detects exactly that and reports it as data: which units,
what each waits for, and who each expected to supply it.

What it does not detect, stated first so the boundary is not mistaken for
a bug:

- **A wait that will end late is not a deadlock.** An operation in the
  kernel completes or errors; a timer fires. Either makes a wait live, and
  a live wait cannot be part of a proved cycle.
- **A cycle through the outside world is invisible.** Two processes
  waiting on each other's sockets are deadlocked and nothing here can see
  it: the substrate's graph ends at the kernel.
- **A wait with a timer in it is never proved deadlocked.** The timer is a
  live half, so the set is not stuck by this definition. That is correct
  and it is also why timeouts hide deadlocks rather than solve them.

## The graph is the wait records

There is no separate graph structure, and no registry. Every parked unit's
wait record already holds both halves of every edge: what it waits for,
and who will end that wait (`design/execution.md`). Traversal reads the
`poster` field of a record and lands in another slot.

This is the choice the plan left open, and the pools are what settle it.
A maintained graph would duplicate the records and could disagree with
them; a scan of the whole pool would cost the population rather than the
question. Following poster links from one unit costs the reachable
subgraph and nothing more.

A full pool scan exists for one purpose only: the diagnostic dump that
answers "what is everything waiting on", which is a human's question and
not the detector's (`design/pool.md`).

## The trigger is wait age

A unit that has been parked longer than a threshold runs the search
itself, on its own worker, before the worker looks for other work. Below
the threshold nothing runs at all.

This is PostgreSQL's arrangement, where `deadlock_timeout` defaults to one
second and the waiter checks only after it expires. The cost in the common
case is zero, and a cycle is found while the rest of the process is busy.

The alternative — declaring deadlock when the whole process falls quiet —
is what `php-src/ext/async` does, and it fails in both directions
(`dev/DECISIONS.md`, 2026-08-12). Three units in a cycle among two hundred
busy ones never make the process quiet, so the cycle is never found. And
quiet does not prove a cycle, which is why that code carries a last-chance
reactor drain against its own false positives.

## Blocked, and the two wait modes

The search computes which units are permanently blocked. It starts by
assuming every parked unit reachable from the suspect is blocked, then
removes the ones that are not, until nothing more can be removed.

A **half** is live when what it waits on will act without help from the
set: an operation the kernel has accepted, an armed timer, a message to a
unit that is runnable or running, a resource with a poster outside the
set.

A **unit** is not blocked when:

| Mode | Not blocked if |
|---|---|
| OR — continues when any half fires | **any** half is live |
| AND — continues when every half fires | **every** half is live |

Removing one unit can make another live, so the removal repeats to a
fixed point. Whatever is still marked blocked at the end, and reachable
from the suspect, is deadlocked.

The two modes are why a single algorithm covers both cases the literature
separates. Under AND alone the answer is a cycle; under OR alone it is a
knot — a set no path leaves. The fixed-point removal computes both without
distinguishing them, because the modes are already in the records.

### A worked case

```
unit 1  AND  [ reply from unit 2 ]                       parked
unit 2  OR   [ mutex M held by unit 1, timer 5 s ]       parked
```

Unit 2 has a live half — the timer — so it is removed. Unit 1's only half
waits on unit 2, which is not blocked, so unit 1 is removed too. No
deadlock is reported, and the report is right: in five seconds unit 2
resumes with a timeout.

Remove the timer and nothing is live: unit 2 waits on a mutex held by unit
1, unit 1 waits on a reply from unit 2, both stay marked, and the pair is
reported.

## Reading a moving system

The search reads slots that other threads are writing. Three rules make
its answer sound, and each closes a specific way of being wrong.

- **Read only parked units.** A unit in `Parked` is not executing, so its
  record is stable while it stays there (`design/execution.md`).
- **Validate every slot twice.** After collecting a slot's fields, re-read
  its generation and its epoch. A change means the unit woke and possibly
  parked again, so the fields may belong to two different waits, and the
  slot is discarded. Without this the search composes a cycle out of two
  waits that never coexisted (`design/pool.md`).
- **Confirm the whole set before reporting.** After the fixed point, walk
  the set again and check every generation and epoch is unchanged since
  the first pass. Any change abandons the search. A deadlock does not
  disappear, so abandoning costs one repetition; reporting a phantom costs
  a victim.

The result is a design that misses a real cycle occasionally and never
invents one. That direction is deliberate: a missed cycle is found on the
next threshold expiry, and a false one kills a healthy unit.

## What is shared with the collector, and what is not

The collector already walks object graphs and already stamps epochs, so
the walker, the header discipline and the epoch bytes are shared rather
than reinvented beside them (`dev/DECISIONS.md`, 2026-08-12).

The trigger and the protocol are not shared, and the reason is sharp: a
deadlocked actor never reaches a message boundary, so it never answers the
collector's handshake. A deadlock stalls mark termination, and a detector
that waited for the collector would be unavailable exactly when it is
needed (`rfc/BACKLOG.md`, 2026-08-12). It needs no handshake of its own,
because it reads only parked units.

## The victim

A proved set is reported, and one member is failed so the rest proceed.
The choice is a policy with a default rather than a law:

1. **Prefer a unit that can be failed at all.** A unit with a live foreign
   frame cannot be unwound (`design/execution.md`), so it is chosen last.
2. **Prefer the youngest wait.** It has done the least work, so failing it
   discards the least.
3. **Prefer a unit whose consumer declared it retryable.** The consumer
   knows what is safe to repeat; the substrate does not.

Failing a unit is cancellation, not a new mechanism: retire its halves,
wake it with a deadlock result carrying the whole set
(`design/cancellation.md`).

**When every member is unkillable** — each parked below a foreign frame —
nothing can be failed and the set is reported without a victim. The
process keeps running with that set stuck, and the diagnostic says so.
This is the honest outcome: the alternative is unwinding through frames
that were not compiled for it, which is undefined and would trade a
diagnosable hang for a crash of unknown shape.

## The report

A detection produces a value, not a log line: the set, each unit's
identity and wait, each poster, and which member was failed. It reaches
the consumer through the same result slot the failed unit resumes on, and
through a diagnostic channel any consumer can poll. `php-src/ext/async`
prints its wait graph to standard output under a debug flag, which means
the information exists exactly where it cannot be used.

## Cost

- **Below the threshold:** nothing. No counter, no list, no barrier on the
  park path beyond the record write that already happens.
- **At the threshold:** one traversal of the subgraph reachable from one
  unit, plus one confirmation pass over the set found. Both are reads.
- **The park path pays** the `poster` field and the epoch, which the record
  carries for other reasons as well.

No figure is claimed here. The traversal is bounded by reachability rather
than by population, and how large that is in practice is a measurement
against real programs that does not exist.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record, the epoch, the parked state | `design/execution.md` |
| slots, generations, and walking | `design/pool.md` |
| retiring halves and waking with a result | `design/cancellation.md` |
| what an operation waiting in the kernel means | `design/reactor.md` |

## Open questions

- **The threshold.** One second is PostgreSQL's default for a database's
  lock waits, and nothing says it fits a runtime whose waits are mostly
  I/O. It is a deployment parameter with no measurement behind it.
- **Repeated searches on the same set.** A stuck set with no killable
  member is re-detected by every member at every threshold expiry, which
  turns a permanent hang into a permanent stream of traversals. A
  suppression window per set is the obvious answer and is not designed.
- **Set-valued posters.** A channel with several registered senders has a
  poster that is a set, and the search treats a live member as making the
  half live. Maintaining that membership cheaply enough for the park path
  is not designed, and it is the case where a scan may beat a traversal
  after all.
- **Whether the confirmation pass needs to be a snapshot.** Two passes
  over a moving system can both succeed while the set changed between
  them in a way neither observed. No scenario has been constructed where
  this yields a false positive, and none has been ruled out either.
