# Channels and Futures

## What they are

A **channel** is a bounded queue of values with two kinds of end. A write end
sends, a read end receives, and both are counted objects that can be cloned
and dropped independently of the channel behind them. A **future** is a
single-assignment cell with the same shape: a promise writes it once, a future
reads it, and either side may exist without the other.

They are one document because the detector treats them as one kind. Both are
*supply* resources: nobody owes a waiter anything, and whether a wait on them
can still be served is a question about who can still reach the write end
(`design/deadlock.md`). The mutex answers such a question with an owner field
and the guard-released semaphore with a holder list and a free count; these two
have nothing to name, and everything below follows from that.

The third supply resource, the actor mailbox, is not here and has no liveness
rule anywhere yet — `design/deadlock.md`'s table has a row for a synchronous
actor call, which is a debtor, and none for a wait on a mailbox. That row is
owed by `design/deadlock.md` and is listed among its open questions rather than
deflected here. Nor is the diagnostic ring of that document here: it shares the
word "channel" and none of the semantics, since it drops on full, blocks
nobody and has no ends.

## Two kinds, because a channel across threads is a different algorithm

A channel that stays inside one thread and a channel that crosses one are two
types, not one type with a flag. TrueAsync arrived at the same split from the
other direction — `Async\Channel` and `Async\ThreadChannel` are separate
classes with separate implementations — and the reason there was the payload: a
cross-thread value has to be deep-copied into persistent memory, which the
common path must not pay for. Here the payload is not the reason. The algorithm
is.

| | Thread-local | Shared |
|---|---|---|
| Reference counts on the ends | non-atomic | atomic |
| Who may hold an end | a coroutine pinned to this thread | any coroutine, and an actor |
| Where the resource lives | the thread's heap | the immortal region |
| Buffer, flags, end counts, queue links | plain reads and writes on one thread | all under the channel's own lock |
| Capacity 0 | rendezvous, a direct handoff | not offered |
| Waking a waiter | inline, on this thread | inline if the waiter is ours, otherwise its owner's intake queue |

The thread-local kind is the default and the cheap one: it takes no lock and
touches no atomic, because everything it reads and writes belongs to the thread
that is running. Where the text below says "under the lock", the thread-local
kind reads it as "in one uninterrupted stretch, with no suspension point
inside", which its single thread gives for free.

**An end of a thread-local channel may be held only by a coroutine pinned to
that thread**, and an actor may hold shared ends alone. An actor moves between
threads at any suspension point (`dev/DECISIONS.md`, 2026-08-13), and a
thread-local end in its hands would have a non-atomic count incremented from
two threads and a buffer written from two threads: a double free rather than a
stale read. Limelight's compiler refuses the assignment; over the C ABI nothing
checks it, and the contract is the consumer's to keep.

**Rendezvous is absent from the shared kind on purpose.** A capacity-0 send
completes only when a receiver takes the value, and across threads that means
the sender parks until another thread runs, which is a round trip per value
rather than a handoff. TrueAsync forbids capacity 0 on its thread channel for
the same reason.

**What may cross is narrower than what may be sent locally.** A value sent
through a shared channel has no recipient at the moment of the send, so the
disposition `rfc/runtime/actors.md` calls *copy* — a deep copy into the
recipient's arena — has no destination and is not available. What remains is
what that document allows without one: a value that may be shared, meaning
immortal or frozen copy-on-write with no non-atomic counts, or a value whose
transfer the compiler proved at its allocation site and which is therefore
moved. An arena-born mutable value may not enter a shared channel.

## What the resource holds, and the two rules the detector imposes

A channel holds its buffer, its capacity, the closed flag with its reason, the
count of live write ends, the count of live read ends, and two queues of
waiters. A future holds the resolved value or the broken flag and the same two
counts.

**No traced reference runs from the resource to an end.** `design/deadlock.md`
requires this and says what breaks without it: with a back-reference, any live
holder of a read end makes the write end reachable through the resource, so
every channel with a live receiver reads as servable and supply resources stop
being detectable at all. Counts are what the resource keeps instead, and a
count is not a reference. The buffered values are the resource's own and are
traced as any object's fields are; the constraint is about the ends.

**The waiter queues hold counted, traced references, and two walks skip
them.** A queue link keeps its waiter alive — `design/execution.md` requires
whoever arms an entry to hold a counted reference, so that a unit cannot be
freed under a wake still on its way, and that requirement is what retired the
generation-and-deferred-reclamation scheme (`dev/DECISIONS.md`, 2026-08-12). A
counted reference the tracer cannot enumerate would be unsound in this memory
model, so the link is traced for the memory mark like any other. What it must
not do is carry the *liveness* mark: otherwise one live holder of a read end
marks every coroutine parked on that channel and none of them can enter the
dead set. The attribution walk skips the same links, or a dead coroutine
holding a read end reaches other coroutines' write ends through the queue, the
dead graph gains edges that do not exist, and the victim is chosen from a
distorted set of sinks. Both exclusions are stated as one flow rule in
`design/deadlock.md`.

Three consequences, and each is a rule for the implementation:

- **A node takes its counted reference before its link is published**, and the
  node carries the unit, the entry index and the epoch.
- **A node has one remover at a time**, under the channel's lock. The reference
  is not dropped at the unlink: it *moves* into the inline wake or into the
  intake payload, and is dropped exactly once by whoever consumes that payload.
  An unlink performed by retirement is idempotent — unlink if linked — and
  drops the reference it found.
- **A missed unlink now leaks rather than dangles**, which is the point of
  keeping the reference counted.

**A write end must be reachable to the tracer, and this is where the design is
not finished.** The detector attributes a write end to the cells of the
coroutine that holds it, and the `walk` hook makes a coroutine's cells
reachable — the two inline wait entries and the spill block
(`dev/DECISIONS.md`, 2026-08-12). A write end held in a *frame* of a stackful
coroutine is in neither: no document here designs enumerating references out of
a coroutine's stack, and for a frame someone else compiled it cannot be done.
Until that closes, a write end the tracer cannot reach must be **registered**,
exactly as a C-ABI holder's is, and a resource whose write ends are not all
traceable or registered is treated as always live. `design/deadlock.md` carries
the same item.

A promise is the future's write end and obeys all of the above. Two rules of
the channel have no promise form and are named where they arise: a future has
no explicit break, and dropping the last future handle while a promise lives
does nothing.

## Capacity

Capacity is fixed when the channel is created and the buffer is allocated once.

- **Bounded, capacity ≥ 1.** A send that finds the buffer full parks; a receive
  that finds it empty parks.
- **Rendezvous, capacity 0**, thread-local only. Every send parks until a
  receiver takes the value, and every receive parks until a sender offers one.
- **Unbounded is not offered.** It removes back-pressure, so a producer's memory
  growth becomes invisible, and it removes the send-side wait, which removes the
  one case in which a send can deadlock.

**A rendezvous value lives in the sender's waiter node**, and the receiver takes
it from there. The channel never owns it. This is what makes closing safe, and
TrueAsync shows the cost of the other shape: it deposits the value into the
channel and then needs a `rendezvous_committed` bit to tell "matched, receiver
has not run yet" from "never matched", because closing must not deliver a value
whose sender has already been failed. With the value in the node there is
nothing to roll back and no bit — a sender failed for any reason clears its own
node while unwinding, and it is the only writer of that field.

**There is no non-blocking send on a rendezvous channel.** Such a call would
have to report success without parking, and with no buffer and no node of its
own it has nowhere to leave the value. TrueAsync offers the call and it behaves
as capacity 1 for that path and as a rendezvous for the blocking one. A caller
that wants to try without blocking uses capacity 1.

## Many ends

Ends are clonable, so there may be many senders and many receivers.

- **A value goes to exactly one receiver.** Delivery is not a broadcast and a
  channel is not a topic.
- **Waiters are woken first in, first out** within each queue. TrueAsync
  advertises Go's `recvq` and `sendq` and does not keep it: its pop takes
  element zero and backfills the hole from the tail, so `[A, B, C]` yields A,
  then C, then B, and a receiver can be skipped indefinitely. An intrusive list
  costs the same and keeps the order.
- **The contract is the order of waking, not the order of delivery**, and the
  difference is worth stating because it is where a promise would otherwise be
  quietly broken. A woken receiver is unlinked before it runs, so an arriving
  receiver may take the value it was woken for — once per value, since the next
  value wakes the next waiter in turn. The overtaken receiver parks again at the
  tail **with the remainder of its deadline** and is not sent back to the start
  of a fresh timeout. Reserving a value for a woken node instead would be
  TrueAsync's committed bit under another name.
- **An arriving caller takes a value inline only when no other waiter is
  queued.** With a queue behind it, it parks at the tail even though the value
  it wanted is in the buffer. The same rule applies to a send against the
  sender queue.

## Closing and breaking

**A channel closes when its last write end is dropped.** This is the rule
`design/deadlock.md` depends on and could not state: the detector marks no
resource, and a resolved coroutine drops its write end while unwinding, which
closes the channel and fails its co-waiters through the ordinary path in the
wake of the same pass.

Four causes, one enum, and the first cause wins — the shape TrueAsync got right
and the part of its channel that never had to be reverted:

| Cause | Reported as |
|---|---|
| the last write end dropped | `no senders` |
| an explicit close on a write end | `closed` |
| the last read end dropped | `no receivers`, to the senders |
| the scope that created the channel ended | `scope ended` |

**A wait proved dead by the detector is not among them.** Closing the channel
would be the mark on a resource that `design/deadlock.md` forbids: a write end
still held by a member of the dead set may belong to a writer the resolution
resurrects, and that writer's legitimate send would then fail against a channel
closed by invention. The `deadlock` cause lives in the **error** the detector's
step 12 stores in the victim's result slot, beside the report handle. A consumer
distinguishes it exactly as before, because it switches on the cause of the
error rather than on a field of the channel. Co-waiters are failed only by the
real consequence: the victim drops its last write end while unwinding and the
channel closes with `no senders`.

**Closedness is read as "the flag, or the corresponding end count at zero"** —
write ends for a receive or an await, read ends for a send — everywhere it is
read, including the detector's liveness table. The decrement that reaches zero
is therefore the moment the channel becomes closed; setting the flag and waking
the waiters may lag behind it harmlessly. Without this the window between the
two is a state in which the buffer is empty, no flag is set and no write end
exists to be marked, and a pass landing inside it reports an invented deadlock
against an ordinary producer's exit.

**An explicit close is a declaration, not a hint**: it closes the channel for
every write end at once, because "no more values" is a fact about the channel
rather than about one holder.

**A receiver drains before it sees the close.** Receive tests emptiness before
closedness, so buffered values are delivered and only the receive that finds
the buffer empty fails.

**The last read end dropping is the symmetric case and is not optional.** The
liveness rule for a send reads the reachability of the read end, so a send whose
read end is gone must fail rather than wait: senders parked on a full buffer are
woken with `no receivers` and later sends fail at once. **The buffered values
are destroyed with that close**, because no receiver can ever appear — a read
end comes only from cloning a live one. Their destructors run outside the
channel's lock, on the thread that performed the close, and a value that may not
be destroyed there is one that may not enter a shared channel in the first
place: the disposition rules above admit only immortal, frozen or moved values,
and a moved value's single owner at that moment is the channel.

**Closing is idempotent** and keeps the reason of its first cause.

**A future resolves once.** A second resolve is an error at the caller rather
than a silent overwrite. Dropping the last promise without resolving *breaks*
the future, breaking is idempotent, and it fails every awaiter with `broken`.

**Scope binding is observation, not ownership.** A channel subscribes to the
close event of the scope that created it and closes with `scope ended` when that
scope ends. The subscription holds a back-pointer so either side may die first
and neither dereferences the other's corpse. Of TrueAsync's three layers against
a stuck channel this is the one that answers lifetime structurally instead of
guessing with a clock, and the only one that was never reverted.

## Registering a wait: decide and publish

Every suspension goes through the one park primitive, and a channel registers
nothing of its own: a wait recorded outside that primitive is invisible to the
detector, and its absence cannot be detected from outside
(`design/execution.md`). The entry names the channel and carries the cancel
handle, the result slot and the fired bit; the kind and the state are read from
the resource and never copied into the entry.

**One critical section decides and publishes, in that order.** Under the
channel's lock: read the state, take a value or deposit one if the rule above
allows it, and otherwise link the node — the link is published only once the
decision to sleep is made. There is no window between the test and the link,
because the lock is the atomicity that an ordering trick would only have
approximated. This also removes the contradiction an earlier form of this
document had, where a node linked before the test made its own queue non-empty
and no caller could ever satisfy an entry inline.

**Every node this wait linked is unlinked when the wait ends without parking.**
Park step 2 arms entries in order, so an entry satisfied inline has to unlink
its own node and the nodes of the entries armed before it — but **only when the
inline satisfaction ends the wait**: under OR mode, or under AND when it was
the last outstanding entry. Under AND with entries still outstanding the
earlier ones stay armed, and retiring them there would leave the wait alive with
nothing able to wake it, which no later signal repairs and which the detector
reads as live because the buffer it waits on is not empty.

**Arming refuses to match another entry of the same wait.** A wait that both
sends and receives on one rendezvous channel would otherwise satisfy itself
inside step 2, or wake itself before step 4 has stored `Parked`. It parks
instead, and the detector reports it as the sink of one that it is.

**A cancel found before the park completes unlinks the nodes.** Park step 4
reads the cancelled bit and stores `Woken` instead of `Parked`; it retires the
entries there, which unlinks every node this wait linked. Without that a
coroutine that is unwinding, or one cancelled while suspending, leaves nodes in
channels it will never look at again, and the reference the queue holds turns a
use-after-free into a leak but does not stop the channel from waking a
coroutine that has since parked on something else.

## The wake carries no value, and the duty it carries instead

**A wake carries no value.** For a buffered channel the value is in the buffer,
for a rendezvous it is in the sender's node, and in neither case does it pass
through the receiver's result slot. The detector therefore reads the buffer's
occupancy — the place the value actually is.

**A wake on a supply resource is a duty, and only consumption discharges it.**
This is the law that replaces the earlier and unworkable rule that a sender
walks the queue until someone accepts. Because the liveness table reads a
non-empty buffer as *live*, a lost supply wake is a hang the detector will never
report, so the obligation has to be an invariant rather than an effort. **Every
path that destroys the last carrier of a duty re-runs the wake.** There are
three carriers and three discharges:

1. **A refusal learned inline** — `wake` returns refused at step 0, 1 or 3 on
   this thread. The waker continues its walk.
2. **A refusal learned on another worker.** The wake was forwarded, so only its
   owner reaches the steps where a refusal is decided; that owner re-runs the
   wake when it drains its intake queue. **The intake payload of a supply wake
   therefore carries the resource** beside the unit, the entry index, the epoch
   and the result. Without it the refusing owner cannot know whose queue to walk:
   on an epoch mismatch the wait record already describes a different wait and
   its resource field belongs to that one.
3. **A wake accepted and never consumed** — a cancel that found the unit in
   `Woken`, a select arm returning without taking, a wake over a value someone
   else took. The woken unit discharges the duty itself, on its own thread,
   inside the substrate's receive, send or select procedure, before the outcome
   escapes to the caller. A unit torn down without resuming does not exist: a
   unit always ends itself (`design/execution.md`), so that frame always runs.

**Re-running the wake is one procedure everywhere:** take the resource's lock,
and if the state gives cause — a value present, space free, or closed — and a
waiter is linked, unlink one node and wake it; otherwise do nothing. Taking a
value frees space and raises the same duty towards the sender queue, so the
symmetry is complete. A debug build counts outstanding duties per resource
against its state.

**`wake`'s answer is three-valued: accepted, refused, forwarded.** Only an
inline refusal drives the walker's loop. Forwarded means the duty travelled with
the payload, and its fate belongs to the owner — which is why the payload needs
the resource. A resource that names a debtor ignores the answer, as it always
has.

The residual race is a spurious wake: a forwarded wake still in flight while a
re-run wakes a second waiter. It costs one park that is paid for anyway, with
the deadline carried across, and a "wake in flight" bit to remove it would buy a
new race for the price of a word.

**A channel's cancel handle unlinks the waiter node.** `design/cancellation.md`
carries the row; unlinking is idempotent, local, and cannot fail, and it
discharges any duty the node was carrying before the cancel propagates.

## The channel's lock is a leaf

**Nothing foreign executes under a channel's lock.** Held under it: the buffer,
the flags, the end counts and the queue links. Forbidden under it: any step of
`wake`, another channel's lock, the ready set's mutex, a write into an intake
queue, a wake-descriptor signal or any other system call, a value's destructor,
and consumer code of any kind. Channel locks do not nest, in either direction.

A send that must wake a waiter runs in this order:

1. Take the channel's lock.
2. Closed — release, and fail with the reason.
3. The sender queue is not empty, or the buffer is full — link this sender's
   node and release, then continue into the park protocol.
4. Put the value in the buffer.
5. The receiver queue is not empty — unlink the head node, taking its reference
   and its payload.
6. Release the lock.
7. If a node was taken, call `wake` outside the lock. Dispatch names this thread
   and its five steps run here, retirement included — retirement takes other
   channels' locks and this one is no longer held, which is what removes both
   the ABBA between two workers and the re-entrant case of an OR wait with two
   entries on one channel. Dispatch names another worker: the payload, carrying
   the resource, goes to that worker's intake queue and its descriptor is rung,
   both outside the lock. Refused: return to step 1 as the next iteration of the
   walk.
8. The send succeeded at step 4 regardless of what became of the wake.

Closing follows the same shape: under the lock, unlink every node into a local
list and set the flag; wake them and destroy any buffered values after the lock
is released.

## Select, and the entry that loses

An OR wait over several channels is the select of this substrate, and the hard
part is the loser. **A losing entry must leave the channel's queue before the
wait's memory can be reused**, which is what retirement does and why unlinking
is an obligation rather than a courtesy. TrueAsync identifies the same problem
and solves it by overriding the dispose handler of the future its `recvAsync`
returns, so that a dropped arm dequeues itself.

An OR wait over two receives on the same channel is legal and one of the two
wins; the loser's node is unlinked by retirement, outside the channel's lock,
which is exactly why step 7 above forbids calling `wake` while holding it.

**The waiter node is a per-wait node in the entry's storage**, inline for a wait
of one or two entries and in the waker's spill block beyond that
(`dev/DECISIONS.md`, 2026-08-12). It is not a single intrusive node reused
across waits: with reuse, a wake refused on the epoch would unlink the node of
the *next* wait on the same channel, leaving a live waiter with no queue entry
and a value in the buffer that wakes nobody. The teardown obligation is what
keeps the spill block safe to resize, since every node it holds is unlinked
before the block is freed.

## Crossing a thread boundary

A shared channel is the only kind that crosses. The value is in the buffer
before any wake exists, and the wake goes inline if the waiter belongs to this
thread and to the owner's intake queue otherwise, because a coroutine is touched
only by its own thread (`design/execution.md`); the intake queue itself is
the scheduler's (`dev/DECISIONS.md`, 2026-08-13).

**A same-thread send applies the wake inline** rather than posting to its own
intake queue, which also keeps the duty on this thread where the walk can
continue. Applying a wake earlier is the safe direction: a conditional
resolution from the detector is dropped when the winner or the fired bits have
moved, so an earlier wake can only cause a resolution to be discarded, never a
healthy waiter to be failed.

**The served set keeps both of its forms.** `design/deadlock.md` collects, beside
its two marks, the resources a pending intake entry would move — a deposit for a
receive or an await, a take for a send — and the liveness table's "or the
channel is served" clause is what reads it. An earlier form of this document
argued that a deposit needs no such form because occupancy already shows the
value; that argument was wrong in the same way for a take, and it ignored the
resolutions and forwarded wakes that ride the same queue. The set stays as that
document defines it.

## Cancellation while parked

A cancel is delivered through the state word and retires the entries
(`design/cancellation.md`). For a channel that means the node leaves the queue
and any duty it carried is re-run before the cancel propagates; there is no
half-delivered value to account for on either side, because a wake carries none.

**A cancelled sender under a rendezvous keeps its value.** It lived in the
sender's own node, so unwinding takes it along, and the duty raised for it is
discharged by the same rule.

**A send while unwinding is trapped only under cooperative cancellation.** Such
a coroutine may park again, so a send on a full channel whose read end is held
by live code that never reads waits forever, and the detector cannot report it —
reachability proves that a value can be taken, never that it will be. Under
final cancellation every park fails, so cleanup cannot be trapped at all. A
cleanup path that must not block uses a non-blocking send on a buffered channel.

## What the detector reads, and under what discipline

The liveness table of `design/deadlock.md` asks three questions of these
resources, and this document owes it the observables to answer them:

| Wait | Live when |
|---|---|
| channel receive | the buffer is non-empty, or the channel is served, or the channel is closed, or the write end is L-marked |
| channel send | the buffer has space, or the channel is served by a pending take, or the channel is closed, or the read end is L-marked |
| future await | it is resolved or broken, or the future is served, or the promise is L-marked |

L-marked is the rule, not "reachable from something that exists": a write end
held in the cells of a parked, undecided coroutine is reachable from it and is
deliberately not L-marked, which is how a supply cycle of two parked coroutines
is detected. "Closed" is read as the flag or the end count at zero, as above.

What a resource must therefore expose: occupancy, remaining space, the closed
flag with its reason, both end counts, a future's resolved and broken flags, and
the write end, read end and promise as distinct traced objects. The kind is read
from the resource and never copied into a wait entry.

**A verdict of death against one of these waits is re-validated on the
resource's fields, not only on the coroutine's record.** `design/deadlock.md`
extends its validation step to occupancy, the flags, both counts and a future's
resolved and broken state, and its owner-side step re-reads the same fields
before applying a resolution. Any movement drops the whole weakly-connected dead
component, exactly as a movement of the record does. Without that extension a
decrement landing inside the pass survives validation and the report published
ahead of confirmation is a lie even under `report-only`.

**A cross-thread read of these fields is a multi-word snapshot**, and the
collector takes it from its own thread while another thread writes. Whether the
discipline it has — a block snapshot and a re-read of recorded cells in a later
phase — extends from single counted cells to a pair of counts, an occupancy and
a flag-with-reason is asserted and unverified; `design/deadlock.md` records it as
an open item and this document adds the resource fields it now depends on.

**A waiter node's payload is one counted cell**, written once by its owner and
cleared once by whoever takes it, both on the owner's thread. That is the shape
the re-read discipline was built for, and it is why the rendezvous value lives in
the node rather than in the wait record, whose entries the detector snapshots
wholesale.

## The semaphore's release API

`design/deadlock.md` files this question here, because its debtor branch holds
only if a permit is released by dropping a guard. **The substrate's semaphore
therefore has no bare post.** `acquire` returns a permit guard, dropping the
guard releases the permit, and no call releases a permit that was never
acquired. That is what ties a release to an acquisition and what keeps the
holder list truthful enough to name a debtor.

Anything reached over the C ABI counts as postable regardless of what it
claims, because its discipline is not ours to see, and a postable semaphore is
a supply resource: live when its free permits cover the request, or it is
served, or the semaphore object is L-marked — for one reached over the C
ABI, its entry in the registered handle table. A semaphore created with zero permits and
posted by a producer that never acquired anything has no holders at all, so
calling it a debtor would report its waiter deadlocked while the producer runs.
A post raises the same duty a deposit does, and discharges it by the same rule.

## Over the C ABI

A consumer may build a channel of its own, and the substrate cannot inspect it.
What it takes on is the contract of its kind: **its write ends live in the
registered handle table**, which is what makes them reachable from a liveness
root, and the resource answers occupancy and closure truthfully. A resource
that adopts neither is treated as always live, so the default costs detection
rather than correctness.

**A write end of a substrate channel handed to a foreign holder is registered
by the substrate at the hand-out**, and unregistered when the holder returns or
drops it. Without that the end sits in memory the collector does not walk while
the channel remains inside the detection regime, and its receiver is failed with
a deadlock that was invented. This is the same hole as a write end in a stackful
frame and it has the same closure.

Nothing checks the foreign side of any of this, and the detector's promise never
to invent a deadlock rests on it. The registered handle table itself, its
register and unregister calls, and how a foreign holder dropping a write end
becomes visible, are named as undesigned in `design/deadlock.md` and stay
undesigned here.

The surface a foreign caller needs is smaller than the PHP one and its
distinctions are the same. Receive takes an output slot and an optional
cancellation event, and returns three outcomes rather than two: a value; a
closed channel, with the reason; or the cancellation fired, with no error at
all. A null output slot waits without consuming, which is what an OR wait over
several channels needs — and because such a caller consumes nothing, it
discharges the duty by the rule above before it returns, and loops, since the
value may be gone by its next call. This is TrueAsync's
`zend_channel_receive_t` contract, specified there before it was needed; over a
C ABI it becomes a status code and an out-parameter, because exceptions do not
cross.

## Decided elsewhere

| Question | Document |
|---|---|
| how a wait is registered, and the wake protocol | `design/execution.md` |
| retiring an entry, and cancel while parked | `design/cancellation.md` |
| the intake queue, and how a worker drains it | `dev/ARCHITECTURE.md`, `dev/DECISIONS.md` 2026-08-13 |
| the liveness fixpoint, the served set, resolution by exception | `design/deadlock.md` |
| what may cross a thread boundary | `rfc/runtime/actors.md` |
| why a coroutine is touched only by its own thread | `dev/DECISIONS.md`, 2026-08-12 |

## Open questions

- **A write end the tracer cannot reach.** Registration closes it for a foreign
  holder and for a hand-out over the C ABI, but a write end in the frame of a
  stackful coroutine needs either stack maps from `ll-model` or the same
  registration, and neither is designed. Until one is, a channel whose write
  ends are not all traceable or registered is always live, which costs
  detection for the most ordinary producer there is.
- **The intake queue's ordering contract** is still owed by the scheduler
  (`dev/DECISIONS.md`, 2026-08-13), and this document adds to what rests on it: the order in
  which waiters on different workers are woken, and now a forwarded wake's duty.
  The owner-side re-read above removes the half of the burden that concerned a
  deposit ahead of a resolution; the served set still needs one scan of the
  queues before the fixpoint.
- **Whether the two kinds share one API.** They share the semantics of this
  document and differ beneath it, so one generic surface is possible and would
  hide which of the two a caller pays for. TrueAsync kept them visibly
  separate; whether we do is undecided.
- **Whether a future hands out a value or a reference.** A reference avoids a
  copy per awaiter and makes the resolved cell's lifetime the future's; a value
  copies once per awaiter. With several awaiters this is a semantic choice
  rather than an optimisation.
- **Back-pressure beyond parking.** A bounded channel parks a sender, which is
  back-pressure for a coroutine and nothing for a producer that never blocks —
  a timer callback, a completion handler. `rfc/BACKLOG.md` carries the same
  question for the actor mailbox.
- **What a consumer over the C ABI binds a channel to**, if anything, given
  that scope binding is Limelight's and the substrate sees only a close event.
- **The scope subscription against the counted-and-traced rule.** It is a link
  from a scope to a channel and back; whether either half must be counted, and
  what the liveness mark does along it, has not been checked the way the waiter
  queues now have been.
