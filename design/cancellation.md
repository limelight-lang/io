# Cancellation

## What cancellation is

Cancellation ends a wait early. It does not kill a unit and it does not
take memory away from the kernel. Those two mistakes are what the design
is arranged against.

**A unit always unwinds itself.** A cancel makes its current wait finish
with a cancelled result; the unit resumes, observes it, and unwinds from
its own suspension point. Nothing unwinds a unit from another thread,
because the frames below the suspension point may be foreign and their
cleanup is not ours to run (`design/execution.md`).

## Cancelling a unit is a wake

A cancel is delivered through the state word, not through an entry. This is
the whole of the fix for two failures the first version of this document
had: a cancelled unit parked on an AND wait never resumed, because the
ordinary wake path decrements `remaining` and returns without waking
until it reaches zero; and a cancel racing with a park was lost, because
it validated an epoch the unit was in the middle of replacing.

**The state word carries a cancelled bit**, and the canceller sets it with
a compare-and-swap on the same word the parking protocol already uses
(`design/execution.md`):

| State found | Transition | Effect |
|---|---|---|
| `Running` | set the bit | the unit's next park attempt fails and it resumes cancelled |
| `Parking` | `Parking → Woken`, bit set | the worker's failed swap enqueues, as for any wake |
| `Parked` | `Parked → Woken`, bit set | the canceller enqueues |
| `Woken` | set the bit | the unit is already owed a slot; it resumes cancelled |
| free, or generation changed | none | the unit is gone; the request reports so |

The transition is one atomic operation on one word, so there is no window
between validating the occupant and marking it, and no epoch to race
against.

**Whoever wins the transition bumps the epoch and retires the entries.**
Bumping the epoch is what makes every armed entry stale, so no other waker
can also claim the wait: the cancel is the winner regardless of mode, and
an AND wait's counter is never consulted. Retirement happens once,
performed by the winner, so cancel handles are called exactly once and
need not be idempotent.

**The parking protocol checks the bit at one place**: the worker's
`Parking → Parked` swap, which fails if the bit is set exactly as it fails
on `Woken`. A unit that begins parking after the bit was set therefore
does not sleep, and the check needs no new ordering because it is the
swap that was already there.

### What the requester learns

A cancel request returns one of four answers, and the deadlock victim
policy needs all four (`design/deadlock.md`):

| Answer | Meaning |
|---|---|
| delivered | the unit will resume cancelled |
| already finished | the generation had changed |
| pinned | the unit is below a live foreign frame; the bit is set, and it takes effect only if the frame returns |
| not deliverable | the unit is parked below a foreign frame that cannot return; nothing more will happen |

Without these the detector cancels a victim, learns nothing, re-detects
the same set, and picks the same victim forever.

### Two levels, and what the bit means afterwards

**Cooperative** cancellation clears the bit when the cancelled result is
delivered. The unit may park again while unwinding, which it must: closing
a TLS session, releasing a remote lock, and flushing a log are all I/O
that happens during cleanup.

**Final** cancellation leaves the bit set. Every subsequent park fails
immediately with a cancelled result, so unwinding cannot block, and a
consumer that catches and discards the first cancellation cannot continue.
Shutdown and an unresponsive victim use this level; ordinary cancellation
does not.

A consumer that swallows a cooperative cancellation has un-cancelled its
unit, which is deliberate — a `catch` block in PHP that handles the
cancellation exception is entitled to continue. A supervisor that means
otherwise escalates to final.

## Retiring an entry

Every entry in a wait record carries an opaque cancel handle, installed by
whoever armed that entry. Retiring is calling it, and what it does depends
on what armed it:

- **A timer** is removed from the wheel. Synchronous.
- **A message to another actor** is marked so the reply is discarded on
  arrival. There is nothing to recall.
- **A kernel operation** is two phases, below.

**Retirement is asynchronous and a retired entry may still fire.** What
makes that harmless is the epoch bump performed by whoever ended the wait:
a late completion validates against the record's current epoch, finds it
moved, and returns (`design/execution.md`). The epoch moves at the moment
the wait ends, not at the unit's next park, which is why the bump belongs
to the winner rather than to the next parking.

## Cancelling a kernel operation is two phases

A submitted operation owns its buffer until the kernel says it is done.
Cancelling asks the kernel to finish sooner; it does not withdraw the
submission and it does not return the buffer.

1. **Request.** Submit a cancel naming the operation by its handle. On
   io_uring the target is matched by `user_data`, which carries the
   operation's handle including its generation, so a cancel cannot match
   an operation that took the same slot later. On Windows `CancelIoEx`
   matches an `OVERLAPPED` address, which is the operation slot's own, and
   the slot is therefore held until the cancel completes as well as until
   the original does. `CancelIoEx` cancels regardless of which thread
   issued the operation, which is what the design needs: the canceller is
   on another thread by construction.
2. **Wait for what the kernel still owes.** The operation slot counts
   owed completions (`design/pool.md`), and cancelling adds the cancel's
   own. The original arrives whether the cancel succeeded, raced and lost,
   or found nothing, and it may carry the real result, because a cancel
   arriving after the data cancels nothing.
3. **Release when the owed count reaches zero**, and not before. For a
   zero-copy send that is the notification and not the result: the result
   says the send happened, the notification says the buffer may be reused
   (`design/reactor.md`). Releasing on the result hands a buffer back
   while the network card may still be reading it, and the next occupant's
   bytes go out on the wire.

For a buffer the kernel provided from a ring, release means returning it
to the ring rather than to the pool, and a completion flagged as retaining
the buffer does not release anything at all.

### Submitting a cancel from the wrong thread

Only the worker that owns a submission queue may write to it
(`design/reactor.md`), and retirement runs on whichever thread ended the
wait. A cancel from another thread is posted to the owning worker's intake
queue and submitted on its next turn, so the latency between retiring an
entry and the kernel seeing the cancel is bounded by one scheduler turn of
that worker rather than by nothing.

`IORING_REGISTER_SYNC_CANCEL` answers inline and therefore blocks the
thread that issues it. It is used at shutdown, on a thread that has
nothing else to do, and nowhere else.

### Inline outcomes

An operation that fails or completes synchronously at submission produces
no completion on either completion backend — `IOSQE_CQE_SKIP_SUCCESS` on
io_uring, a synchronous error or `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS` on
Windows. The submission path resolves those before the operation slot
enters the owed-count protocol, so phase 2 never waits for a completion
that was never queued.

## The readiness backends are not free

The first version of this document said there is nothing to cancel on
kqueue and epoll because the syscall was never issued. That is wrong in
three cases, and they are the cases that matter on the platforms which
have only readiness backends — macOS and iOS on kqueue, Android on epoll,
where io_uring is blocked.

| Case | What is outstanding | What cancellation must do |
|---|---|---|
| registered, not yet signalled | nothing | deregister; the buffer is free at once |
| signalled, syscall in progress | the reactor thread is inside `read` writing the buffer | wait for that syscall to return; deregistration retracts nothing |
| file I/O on the thread pool | a blocking syscall against a possibly hung mount | wait for the pool thread; there is no cancel |
| `connect` in progress | an in-kernel handshake | close the descriptor; deregistration does not stop it |

So the two-phase shape is the same everywhere, and only the first row
completes phase 2 inline. A readiness backend's "completion" is the
reactor's own record that the syscall returned, which is what phase 2
waits for.

## Timeouts

A timeout is not a cancellation mechanism; it is an OR wait with a timer
in it (`design/execution.md`). The timer firing ends the wait with a
timeout result and retires the other entries like any winner.

The distinction is what keeps timeouts free of their own machinery: no
timeout state, no per-operation timeout list, no second path through the
reactor.

## When a cancelled operation never completes

Phase 2 has no timeout, because the kernel offers none. An operation
against a hung mount stays outstanding as long as the mount does.

- **The resources are accounted, not leaked.** The operation slot stays
  live and its buffer stays pinned, both visible in the pools, so a walk
  names them (`design/pool.md`).
- **The unit does not wait for them.** It resumed on the cancelled result
  the moment the state word moved, and it may complete and have its slot
  reused; the operation holds what it needs on its own.
- **There is no forced reclamation.** Unmapping a buffer the kernel may
  write into is not available. Enough stuck operations exhaust the buffer
  pool and the process stops accepting work — bounded and reportable,
  rather than corrupt.

## Shutdown

Ending a scope, an actor, or the process is final cancellation applied to
a set, and it inherits every limit above.

**Exit does not wait for cancelled operations, and cannot fully avoid
waiting either.** Tearing down an io_uring ring waits for its in-flight
requests, and Windows process teardown waits for drivers to complete
cancelled IRPs. An operation stuck in an uninterruptible stage therefore
delays exit no matter what this design does. The honest statement is that
shutdown waits for units and abandons operations, and that the kernel may
still hold the process in teardown afterwards.

## Decided elsewhere

| Question | Document |
|---|---|
| the state word, the epoch, the park order, `wake` | `design/execution.md` |
| operation slots and their owed-completion count | `design/pool.md` |
| how operations are submitted and completed | `design/reactor.md` |
| who is chosen as a victim | `design/deadlock.md` |

## Open questions

- **A bound on stuck operations.** Accounted, not bounded. A watermark
  that refuses new submissions on a descriptor with too many outstanding
  cancels would turn slow exhaustion into an early error.
- **Cancelling a multishot operation.** Its owed count is unknown until
  the series ends, so "release when the count reaches zero" needs a rule
  for a series that a cancel truncates, and an ordering against
  completions already queued in the socket's slot
  (`design/reactor.md`).
- **`IORING_REGISTER_SYNC_CANCEL`'s kernel floor**, which matters on
  Android and on older long-term kernels, is not established here.
- **A created-but-unmounted unit** holds none of the four states and has
  no wait record, so the cancelled bit is all a cancel can set. It is
  mounted, resumes, and unwinds immediately, which wastes a mount and has
  not been compared against scanning the run queue.
