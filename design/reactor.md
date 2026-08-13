# The Reactor

## What the reactor is

The reactor submits operations to the operating system and delivers their
completions to the units waiting on them. It is one crate, `io-reactor`,
depending on `io-core` and never the reverse (`dev/ARCHITECTURE.md`).

## The API is completion-first

An operation names what to do, what memory to do it with, and who to wake.
It does not report readiness.

```
io_read (waiter, fd, buffer, offset) -> operation handle
io_write(waiter, fd, buffer, offset) -> operation handle
```

The result — bytes transferred, or an error — arrives in the wait record's
result slot when the operation completes.

**Why this direction.** io_uring and Windows IOCP are completion-based,
and readiness backends emulate completion by issuing the syscall when the
descriptor signals. The reverse emulation costs two round trips where the
native shape costs one. Regular files sharpen it: `epoll_ctl` rejects a
regular file outright with `EPERM`, and kqueue's read filter on a vnode
reports bytes-to-EOF while a read still blocks on a page-cache miss. A
readiness API cannot express file I/O, which is why every runtime built on
one has a thread pool behind it.

io_uring is a backend and not the foundation (`dev/DECISIONS.md`,
2026-08-12): Android blocks it from applications through seccomp-bpf,
ChromeOS disables it, and the default seccomp profiles of Docker and
containerd reject it.

## One ring per worker

Each worker owns its own submission and completion queues. A unit submits
on the worker it is mounted on, and that worker drains its own
completions.

Anything crossing a worker boundary — a cancel from another thread, an
operation whose unit migrated — is posted to the owner's intake queue and
submitted on its next turn; the owner is woken through its wake
descriptor if it is blocked in the kernel. A single shared ring was the
alternative and it hangs: without a polling thread, `io_uring_enter`
submits only what is queued at the moment of the call, so entries written
by another worker while the owner sleeps are never submitted at all.

Submissions are batched and flushed once per turn. Two failures have to be
handled rather than assumed away: the submission ring can fill inside one
turn, and `io_uring_enter` can return `-EBUSY` when the completion ring is
overflowing. Both mean "submit what fits, drain completions, retry", and
neither may drop an operation on the floor.

## Three buffer contracts

The buffer is part of the contract, because the kernel owns it from
submission until it says otherwise. The three forms are three ownership
rules, each mapping to a different kernel mechanism. None accepts a stack
address (`dev/DECISIONS.md`, 2026-08-12).

**In all three, the buffer belongs to the operation and not to the
caller.** The operation slot pins it and releases it when the kernel owes
nothing more (`design/pool.md`). A caller never returns a buffer it
handed to an operation, and in particular the loser of an `await` with a
timeout returns nothing: its read is still in the kernel, and the
operation that owns the buffer is what remembers.

### Contract 1 — the caller names a buffer

The caller takes a buffer from the pool and passes it. Native everywhere:
`IORING_OP_READ` and `WRITE`, `ReadFile` and `WriteFile` with an
OVERLAPPED, or the plain syscall issued on readiness.

### Contract 2 — a vector, or a registered pool

Several buffers for a scatter or gather, and the same shape with the pool
registered ahead of time. `IORING_REGISTER_BUFFERS` pins the pages once
and operations name a buffer by index, which is possible only because
slabs do not move (`design/pool.md`). Windows has registered buffers
through RIO; whether to use it is a scoping decision this project has not
made, so the Windows backend starts with a `WSABUF` array. Readiness
backends issue `readv` or `writev`.

### Contract 3 — the kernel names the buffer

The caller registers a ring of buffers and submits an operation without
one; the kernel picks a buffer when data arrives and reports which.

This is `IORING_REGISTER_PBUF_RING`, Linux 5.19. The ring holds a
power-of-two number of entries, at most 32 768. `IOU_PBUF_RING_INC`
(6.12) lets a buffer be consumed incrementally, and only for such a ring
does a completion carry `IORING_CQE_F_BUF_MORE`, meaning the kernel is not
finished with the buffer and it may not be returned yet.

**This is the contract that scales to idle connections.** Under contracts
1 and 2 a socket waiting for data holds a buffer for as long as it waits.
Under contract 3 it holds none, and the ring is sized for the traffic
rather than for the connection count.

Emulation elsewhere:

- **IOCP** reaches the same property by a different route: a receive
  posted with a zero-length buffer completes when data arrives without
  locking any memory, and the real buffer is taken then. This is the
  standard Windows idiom for idle connections and the backend uses it,
  because taking a buffer at submission would put a hundred thousand
  buffers behind a hundred thousand idle sockets on one of the two
  first-class platforms.
- **kqueue and epoll** take a buffer when the descriptor signals, which is
  the readiness world's native shape and what libuv's allocation callback
  does.

## Multishot

A multishot operation is submitted once and completes repeatedly: `accept`
yielding descriptors, `recv` yielding chunks. Each completion carries
`IORING_CQE_F_MORE` while more are coming.

**A multishot operation targets a socket, not a unit.** Its completions
append to a queue owned by the socket slot; they are not validated against
a unit's epoch, because the unit re-parks between chunks and any
unit-scoped validation would discard everything after the first. The
queue's entries come from a pool of completion records, with head and tail
in the socket slot, so a fixed-size slot holds an unbounded series.

**Parking on the queue is an ordinary park with a recheck, and the recheck
belongs to whoever writes the queue-waiter cell.** Without it a completion
landing between the drain and the park sees no waiter, appends, and nobody
ever looks again — the unit sleeps over a full queue.

**The cell belongs to the worker whose ring carries the multishot**, which
is the worker that drains these completions, so one thread writes it, reads
it and empties it (`design/pool.md`). Two paths follow from that:

- **The ring's own worker** writes the cell at park and re-reads the queue
  inline. Non-empty means it empties the cell again and abandons the
  suspension, which is the ordinary inline satisfaction of
  `design/execution.md`.
- **Any other worker** posts the publish to the owner's intake queue and
  suspends without reading the queue at all, that being another worker's
  memory. The owner writes the cell when it drains the entry, re-reads the
  queue, and on non-empty moves the reference out of the cell into an
  ordinary `wake`, which dispatch forwards back to the unit's worker.

The append reads the cell after appending and the publish reads the queue
after writing the cell, both on the one thread that owns the slot, so
whichever runs second sees the other's first half. A unit's own remote
re-read would order nothing, because it precedes the publish it exists to
defend. The intake queue carrying the publish is one more consumer of the
ordering contract the scheduler owes for that queue (`dev/DECISIONS.md`,
2026-08-13): the queue is `sched`'s, and this document posts into it.

**The series ends routinely, and the reactor re-arms it.** Multishot
terminates when the provided-buffer ring runs dry with `-ENOBUFS` and when
a completion overflows the completion ring — both at peak load, not at
idle. The terminal completion carries an error and no buffer; it is
appended to the queue as an entry of its own, and the reactor resubmits
the operation as soon as a buffer is available. A consumer sees a gap in
the stream, never a silent stall.

**A series that cannot be re-armed is the reactor's to report.** A socket
whose series ended and whose re-arm has no buffer sits in the reactor's
re-arm list with the time the series ended; an entry older than the
threshold is published to the diagnostic channel of `design/deadlock.md`
together with the buffer pool's state, and a counter of starved re-arms
separates a quiet system from one that cannot feed its sockets. The
detector reports none of this: the liveness row for a wait on the queue
reads always live, because a pass cannot prove that no buffer ever frees.

**Draining on death.** When a socket closes or its waiting unit ends,
every queued entry is drained: provided buffers go back to the ring and
accepted descriptors are closed. A multishot accept yields descriptors
rather than buffers, so the two cases need different drains and both are
the socket slot's responsibility. **A close also empties the queue-waiter
cell into a `wake` carrying the close error**, and the liveness row rests on
it: a waiter over a queue whose socket closed is failed by this path and by
nothing the detector does (`design/deadlock.md`).

The queue is bounded by a watermark; above it the reactor stops re-arming
and lets the kernel's own socket buffers apply backpressure. The number is
not chosen.

## Zero-copy

**Send.** `IORING_OP_SEND_ZC` and `SENDMSG_ZC` post the send result and,
**only if that result carries `IORING_CQE_F_MORE`**, a second completion
flagged `IORING_CQE_F_NOTIF` that licenses reuse of the buffer. A send
that fails posts one completion and no notification ever follows.

The operation's owed-completion count is therefore read from the first
completion's flags rather than assumed to be two, and the buffer is
released when the count reaches zero (`design/pool.md`). Assuming two
strands a buffer on every failed send, and under connection churn the pool
drains.

A caller that links a zero-copy send with `IOSQE_IO_LINK` must set
`MSG_WAITALL`: a short send is not an error without it, and the link would
proceed on partial data.

Zero-copy send pins pages and adds a completion, so below some payload
size a copy is cheaper. The threshold is unmeasured, so the API exposes
the mode and does not choose it.

**Receive.** io_uring zero-copy receive arrived in Linux 6.15 and needs
hardware header-data split, flow steering and a dedicated receive queue.
Its memory is a registered area refilled through its own ring, with the
kernel choosing offsets — a fourth ownership shape, not one of the three
above. It is not exposed until there is a reason to add that shape.

**File to socket.** io_uring has no `sendfile`. It has
`IORING_OP_SPLICE` (5.7), and splice moves data only when one end is a
pipe, so the io_uring path is file→pipe→socket: a pipe pair per transfer
and two linked submissions, with the pipe's capacity as a throughput step.
Readiness backends use `sendfile(2)` and Windows uses `TransmitFile`,
both of which are one call. The API exposes one operation, and the
io_uring implementation of it is the expensive one — the opposite of the
usual ranking, and worth measuring before choosing the path per backend.

## Completions

A completion is matched to its operation slot by `user_data`, which
carries the operation's handle including its generation, so a completion
for an operation whose slot was reused cannot be mistaken for the
successor's (`design/pool.md`).

The reactor then calls `wake(waiter, entry, epoch, result)` and does not
touch the wait record itself. Validation and the result store are steps 1
and 2 of waking (`design/execution.md`); doing them in the reactor would
open a window between validating and storing in which the unit could win
another entry and re-park, and the result would land in the wrong wait.

A completion whose operation slot is gone releases what it holds and
returns.

## Cancellation, in one paragraph

Cancelling an operation is submitting a cancel that names it, waiting for
what the kernel still owes, and releasing then.
`IORING_REGISTER_SYNC_CANCEL` is a register call rather than a submitted
operation and produces no completion, which is why it is used only where
a thread can afford to block. On the readiness backends an operation
cancelled before its descriptor signalled was never issued, so the backend
**synthesizes** a completion for it; without that, the release condition
never fires and every timed-out read leaks a buffer on macOS, iOS and
Android. `design/cancellation.md` owns the protocol.

## Kernel feature floors

io_uring is not one capability. A backend that probes only for the ring
will fail later, feature by feature.

| Feature | Linux |
|---|---|
| `IORING_OP_SPLICE` | 5.7 |
| `IORING_REGISTER_PBUF_RING` (contract 3) | 5.19 |
| multishot `recv`, `SEND_ZC`, `IORING_REGISTER_SYNC_CANCEL` | 6.0 |
| `SENDMSG_ZC` | 6.1 |
| `IOU_PBUF_RING_INC` | 6.12 |
| zero-copy receive | 6.15 |

The backend probes each at startup and degrades per feature: contract 3
falls back to contract 1, multishot to repeated single-shot submissions,
zero-copy send to an ordinary send. A distribution on 5.15 runs with the
ring and none of the rest, which is a supported configuration rather than
a failure.

## Backends

| Backend | Platform | Shape | Contract 3 |
|---|---|---|---|
| io_uring | Linux, where permitted and where the feature exists | native completion, multishot, registration | native |
| IOCP | Windows | native completion | zero-length receive, buffer taken at signal |
| kqueue | macOS, iOS | readiness; syscall issued on signal | buffer taken at signal |
| epoll | Linux fallback, Android | readiness; syscall issued on signal | buffer taken at signal |

Regular-file I/O on the readiness backends goes to a thread pool, because
readiness carries no information about a file. The pool belongs to the
backend rather than to the scheduler, and its size is a deployment
parameter.

Windows has an `IoRing` API on Windows 11 with a narrower operation set.
It is not a backend here: IOCP covers the platform, and a second Windows
backend before the first is measured would be work with no number behind
it.

## Errors and short transfers

An operation completes with a byte count or an error; the substrate does
not retry and does not interpret. A short read is a result, not a failure,
and the caller decides.

## Decided elsewhere

| Question | Document |
|---|---|
| how a unit parks and what `wake` does | `design/execution.md` |
| buffers, sockets, operations, and owed completions | `design/pool.md` |
| cancelling, and what the kernel still owes | `design/cancellation.md` |
| what a wait on an operation means to the detector | `design/deadlock.md` |

## Open questions

- **Backend choice at startup**, and what a process does when a probe
  succeeds and a later seccomp tightening makes the ring unusable.
- **Thread pool sizing** for file I/O, and whether one pool serves all
  backends.
- **The zero-copy send threshold**, unmeasured, and whether the substrate
  should pick it at all.
- **Registration lifetime.** Growing a pool by a slab needs a sparse
  registration and an update call, whose kernel floor and cost are not
  established (`design/pool.md`).
- **The re-arm starvation threshold**, a deployment parameter with no
  measurement, like the detector's.
- **The multishot queue watermark**, and what backpressure means on the
  emulating backends, where a level-triggered readiness series never ends
  on its own.
- **`splice` versus `sendfile` per backend**, given that the flagship
  backend has the more expensive path.
