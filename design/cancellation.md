# Cancellation

## What cancellation is

Cancellation ends a wait early. It does not kill a unit, and it does not
take memory away from the kernel. Those are the two mistakes the design
is arranged against, and everything below follows from keeping them
apart.

Three things can be asked to end, and they are different operations:

| Asked to end | Mechanism | Who may ask |
|---|---|---|
| one half of a wait | retire it through its cancel handle | the winner of an OR wait, or a cancel request |
| a unit's current wait | retire every half, then wake with a cancelled result | the consumer, a supervisor, the deadlock victim policy |
| a unit's life | it ends itself after observing the cancelled result | nobody directly |

**Nothing ends a unit from outside.** A cancelled unit resumes, sees a
cancelled result, and unwinds from its own suspension point. That is what
keeps the state machine four states wide (`design/execution.md`) and what
makes cancellation safe in the presence of foreign frames, which cannot
be unwound by anyone else.

## Retiring a half

Every entry in a wait record carries an opaque cancel handle, installed
by whoever armed that half (`design/execution.md`). Retiring is calling
it. What it does depends on what armed the half:

- **A timer** is removed from the timer wheel. Synchronous, and the
  handle is done when it returns.
- **A message to another actor** is marked so the reply is discarded on
  arrival. There is nothing to recall.
- **A kernel operation** is a different animal, and the rest of this
  document is about it.

### Retirement is asynchronous, and that is the invariant

A retired half may still fire. The winner of an `await` with a timeout
resumes while the losing read is still in the kernel, and no ordering
between them is available. The epoch check in `wake` is what makes this
harmless: the record's epoch moved when the unit parked again, so the
late completion validates and returns (`design/execution.md`).

Every part of the design that could have depended on "retired means
finished" instead depends on the epoch. That includes stack release,
which does not wait for anything, because nothing the kernel touches
lives on a stack (`dev/DECISIONS.md`, 2026-08-12).

## Cancelling a kernel operation is two phases

A submitted operation owns its buffer until its completion arrives.
Cancelling asks the kernel to finish it sooner; it does not withdraw the
submission and it does not return the buffer.

1. **Request.** Submit a cancel naming the operation. On io_uring this is
   `IORING_OP_ASYNC_CANCEL`, or `IORING_REGISTER_SYNC_CANCEL` where an
   inline answer is wanted. On Windows it is `CancelIoEx`. On the
   readiness backends there is nothing to cancel: the syscall was never
   issued, so the descriptor is simply deregistered and the buffer is
   free immediately.
2. **Wait for the original completion.** It arrives whether the cancel
   succeeded, raced and lost, or found nothing. It may report the
   operation's real result, because a cancel that arrives after the data
   did cancels nothing.
3. **Release on the original completion, never on the cancel's.** The
   operation slot is released there, and the buffer with it
   (`design/pool.md`).

The cancel itself completes separately and carries only whether it found
its target. That completion releases the cancel's own slot and nothing
else.

**The failure this ordering prevents:** releasing the buffer when the
cancel completes hands pool memory back while the kernel may still be
writing into it. The next taker of that buffer gets bytes from a socket
it never read.

## Cancelling a unit

A cancel request against a unit does three things in order:

1. **Mark the unit cancelled** in its slot, so a unit that is about to
   park sees the mark and does not.
2. **Retire every half** of its current wait, if it is parked.
3. **Wake it with a cancelled result**, through the same `wake` call as
   any completion, with the same epoch validation.

The unit resumes, observes the result, and unwinds. Where it cannot
unwind — a live foreign frame — it is not cancelled at all: the mark
stays set, and the unit is cancelled at its next suspension point above
the foreign frame, if it reaches one. A unit parked below a foreign frame
that never returns cannot be ended (`design/execution.md`), and that is a
statement about the world rather than a gap in the design.

**Cancellation is idempotent.** A second request finds the mark set and
returns. A request racing with the unit's own completion finds the
generation changed and does nothing (`design/pool.md`).

## Timeouts

A timeout is not a cancellation mechanism; it is an OR wait with a timer
in it (`design/execution.md`). The timer firing makes the unit resume
with a timeout result, and the read half is retired like any loser.

The distinction matters because it is what keeps timeouts free of their
own machinery: there is no timeout state, no timeout list per operation,
and no second path through the reactor. Whoever wants a bounded wait arms
a timer half.

## When a cancelled operation never completes

The design above waits for the original completion, and there is no
timeout on that wait, because the kernel does not offer one. An operation
against a hung network filesystem can be outstanding for as long as the
mount is hung, and its buffer and operation slot stay held.

What the design does about it, and what it does not:

- **The resources are accounted, not leaked.** The operation slot stays
  live and the buffer stays pinned, both visible in the pools, so a walk
  reports them (`design/pool.md`). A hundred stuck operations are a
  hundred slots a diagnostic can name, not memory that vanished.
- **The unit does not wait for them.** It resumed on the cancelled result
  in phase 1 and may complete, die, and have its slot reused. The
  operation holds what it needs on its own.
- **There is no forced reclamation.** Unmapping a buffer the kernel may
  write into is not available. A process with enough stuck operations
  exhausts its buffer pool and stops accepting work, which is a bounded
  and reportable failure rather than corruption.

## Backend differences

| Backend | Cancel request | Does the original still complete? | Buffer released on |
|---|---|---|---|
| io_uring | `IORING_OP_ASYNC_CANCEL` or `IORING_REGISTER_SYNC_CANCEL` | yes, always | the original completion |
| IOCP | `CancelIoEx` | yes, with `ERROR_OPERATION_ABORTED` or the real result | the original completion |
| kqueue, epoll | deregister the descriptor | there is no original: nothing was submitted | immediately |

The readiness backends make cancellation cheap and the completion
backends make it two-phase, which is the reverse of how the rest of the
substrate ranks them. It is the one place where readiness is the simpler
world, and the API hides the difference by always going through the same
two phases — on kqueue and epoll, phase 2 completes inline.

## Shutdown

Ending a whole scope, an actor, or the process is cancellation applied to
a set, and it inherits every limit above. In particular a shutdown does
not wait for cancelled operations to complete before the process exits;
it waits only for the units. An operation still in flight at exit is
abandoned to process teardown, which is safe because the address space
goes with it.

What is not safe is exiting while a registered buffer pool is still
registered with a ring that outlives the process image — which cannot
happen, and is recorded here only so that a future backend with a
kernel-persistent ring does not quietly break it.

## Decided elsewhere

| Question | Document |
|---|---|
| the wait record, the epoch, and `wake` | `design/execution.md` |
| operation slots and what they pin | `design/pool.md` |
| how an operation is submitted and completed | `design/reactor.md` |
| who is chosen as a victim, and what happens if it is unkillable | `design/deadlock.md` |

## Open questions

- **A bound on stuck operations.** The design accounts for them and does
  not bound them. A watermark that refuses new submissions on a
  descriptor with too many outstanding cancels would turn a slow
  exhaustion into an early error, and it is not designed.
- **Whether `IORING_REGISTER_SYNC_CANCEL` is worth a second path.** It
  answers inline, which suits shutdown, and it is a different code path
  from the ordinary cancel. Unmeasured.
- **Cancelling a multishot operation.** It has many completions, so
  "wait for the original completion" needs a definition: presumably the
  completion without `IORING_CQE_F_MORE`, but the ordering against
  queued-but-undelivered completions in the socket's slot
  (`design/reactor.md`) is not worked out.
- **Cancellation of a unit that has not yet been mounted.** The mark is
  in the slot and the handle is in a run queue; nothing dequeues it early,
  so it is mounted, resumes, and immediately unwinds. That wastes a mount
  and is probably fine, but it has not been compared against a queue scan.
