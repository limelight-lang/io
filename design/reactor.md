# The Reactor

## What the reactor is

The reactor submits operations to the operating system and delivers their
completions to the units waiting on them. It is one crate, `io-reactor`,
depending on `io-core` and never the reverse (`dev/ARCHITECTURE.md`). A
completion is delivered by the same `wake(data, half, epoch, result)`
call a stackless unit's waker uses, so the reactor knows nothing about
how a unit suspends (`design/execution.md`).

## The API is completion-first

An operation names what to do, what memory to do it with, and who to wake.
It does not report readiness.

```
io_read (unit_half, fd, buffer, offset) -> submitted
io_write(unit_half, fd, buffer, offset) -> submitted
```

The result — bytes transferred, or an error — arrives in the wait
record's result slot when the operation completes.

**Why this direction and not readiness.** io_uring and Windows IOCP are
completion-based, and readiness backends emulate completion by issuing
the syscall when the descriptor signals. The reverse emulation loses what
io_uring is for: a readiness API on top of it would submit a poll, wake,
and then submit the real operation, paying two round trips where the
native shape pays one. Files make the asymmetry sharper — a regular file
is always "ready" under kqueue and epoll, so a readiness API cannot
express file I/O at all and every runtime built on one ends up with a
thread pool behind it.

io_uring is a backend and not the foundation (`dev/DECISIONS.md`,
2026-08-12): Android blocks it from applications through seccomp-bpf,
ChromeOS disables it, and the default seccomp profiles of Docker and
containerd reject it.

## Three buffer contracts

The buffer is part of the contract, because the kernel owns it from
submission until completion. The three forms are not conveniences; they
are three ownership rules, and each maps to a different kernel mechanism.

None of them accepts a stack address. Everything the kernel touches comes
from the buffer pool (`dev/DECISIONS.md`, 2026-08-12).

### Contract 1 — the caller names a buffer

The caller takes a buffer from the pool and passes it. The kernel owns it
from submission until the completion arrives; the caller may not read,
write, or release it in between, and cancelling does not shorten that.

Native on every backend: `IORING_OP_READ` and `WRITE`, `ReadFile` and
`WriteFile` with an OVERLAPPED, or the plain syscall issued on readiness.

### Contract 2 — the caller names several buffers, or a registered pool

A vector of buffers, for a scatter or gather, and the same shape with the
pool registered ahead of time.

Registration is what makes it worth a separate contract.
`IORING_REGISTER_BUFFERS` pins the pool's pages once, and operations then
name a buffer by index instead of by address, so the kernel skips the
per-operation page lookup. This is only possible because pool slabs do
not move (`design/pool.md`). Windows takes the same shape as a `WSABUF`
array; readiness backends issue `readv` or `writev`.

A consumer whose memory manager cannot promise stable slabs gets contract
2 without registration, which costs the page lookup and nothing else.

### Contract 3 — the kernel names the buffer

The caller registers a ring of buffers and submits an operation without
one. The kernel picks a buffer when data arrives and reports which it
used; the caller reads it and returns it to the ring.

This is `IORING_REGISTER_PBUF_RING`, available since Linux 5.19. The ring
holds a power-of-two number of entries, at most 32 768. The completion
carries the buffer index in its flags, and `IOU_PBUF_RING_INC` lets a
buffer be consumed incrementally rather than whole. `IORING_CQE_BUF_MORE`
says the kernel is not finished with the buffer, so the caller may not
return it yet; for any other provided-buffer type a completion that hands
a buffer back is final.

**This is the contract that scales to idle connections.** Under contracts
1 and 2 a socket waiting for data holds a buffer for as long as it waits,
so a hundred thousand idle connections hold a hundred thousand buffers.
Under contract 3 they hold none, and the ring is sized for the traffic
rather than for the connection count.

It is also what makes multishot possible: one submission that produces
many completions, each with its own buffer.

Emulation elsewhere: on IOCP a buffer is taken from the pool at
submission, so the API contract holds while the mechanism degrades to
contract 1; on kqueue and epoll a buffer is taken when the descriptor
signals, which is exactly what libuv's allocation callback does and is
the readiness world's native shape.

## Multishot

A multishot operation is submitted once and completes repeatedly:
`accept` yielding connections, `recv` yielding datagrams or stream
chunks. Each completion carries `IORING_CQE_F_MORE` while more are
coming, and the absence of that flag ends the series.

Multishot changes what a wait means, and the change belongs here rather
than in `execution.md`. A unit cannot park on a multishot operation the
ordinary way, because a wait record entry is retired when its half fires.
A multishot operation therefore feeds a queue in its socket's slot, and
the unit parks on the queue: the first completion that finds the queue
empty and a unit parked wakes it, and later completions append. This
keeps the wait record's one-shot semantics intact and puts the buffering
where the socket already is.

## Zero-copy

Zero-copy means different mechanisms in each direction and the API says
which one it is rather than promising the property.

**Send.** `IORING_OP_SEND_ZC` and `SENDMSG_ZC` produce **two**
completions: the send result, flagged `IORING_CQE_F_MORE`, and a
notification flagged `IORING_CQE_F_NOTIF` that says the buffer may be
reused. The contract therefore has two events where every other operation
has one, and the buffer returns to the pool on the second. A caller that
links a zero-copy send with `IOSQE_IO_LINK` must set `MSG_WAITALL`,
because a short send is not an error without it and the link would
proceed on partial data.

Zero-copy send is not free at every size: it pins pages and adds a
completion, and below some payload size the copy is cheaper. The
threshold is a measurement nobody here has taken, so the API exposes the
mode and does not choose it.

**Receive.** io_uring zero-copy receive landed in Linux 6.15. It requires
hardware header-data split, flow steering and a receive queue configured
for it, so it is an option a deployment enables rather than a path the
substrate can assume.

**File to socket.** io_uring has no `sendfile` operation; it has
`IORING_OP_SPLICE`, available since 5.7. Readiness backends use
`sendfile(2)`, and Windows uses `TransmitFile`. The API exposes one
operation over the three.

## Backends

| Backend | Platform | Shape | Contract 3 |
|---|---|---|---|
| io_uring | Linux, where permitted | native completion, multishot, registration | native |
| IOCP | Windows | native completion | emulated: buffer taken at submission |
| kqueue | macOS, iOS | readiness; syscall issued on signal | native shape: buffer taken at signal |
| epoll | Linux fallback, Android | readiness; syscall issued on signal | native shape |

Regular-file I/O on the readiness backends goes to a thread pool, because
a file descriptor is always readable and readiness carries no information
about it. That pool is part of the backend rather than of the scheduler,
and its size is a deployment parameter.

Windows has an `IoRing` API on Windows 11 with a submission and
completion ring resembling io_uring, and a narrower operation set. It is
not a backend here yet: IOCP covers the platform, and adding a second
Windows backend before the first is measured would be work with no number
behind it.

## Submission and completion

Submissions are batched: an operation is written into the backend's queue
and the queue is flushed once per scheduler turn rather than once per
operation. A unit that submits and parks does not force a flush, because
its own park is what returns control to the worker, and the worker
flushes before it waits.

Completions are drained on the worker that owns the backend's queue. For
each completion the reactor finds the operation slot, validates the
generation and epoch against the waiting unit, writes the result, and
calls `wake` (`design/pool.md`, `design/execution.md`). A completion
whose validation fails is a completion for something that no longer
exists, and it releases its resources and returns.

## Errors, short transfers, and cancellation

An operation completes with a result that is a byte count or an error;
the substrate does not retry and does not interpret. A short read is a
result, not a failure, and the caller decides.

Cancellation is submitted like any other operation
(`IORING_OP_ASYNC_CANCEL`, or `IORING_REGISTER_SYNC_CANCEL` where an
inline answer is wanted) and completes like any other. The cancelled
operation still completes, separately, and its buffer is released on that
completion rather than on the cancel's. `design/cancellation.md` owns the
protocol.

## Decided elsewhere

| Question | Document |
|---|---|
| how a unit parks and what `wake` does | `design/execution.md` |
| where buffers, sockets and operations live | `design/pool.md` |
| what happens to an operation when its unit is cancelled | `design/cancellation.md` |
| what a wait on an operation means to the detector | `design/deadlock.md` |

## Open questions

- **Which backend a deployment gets, and how it is chosen.** Probing
  io_uring at startup and falling back to epoll is the obvious answer,
  but a probe that succeeds under a permissive seccomp profile and fails
  later under a tightened one gives a running process no path back.
- **Thread pool sizing for file I/O** on the readiness backends, and
  whether one pool serves all backends or each owns its own.
- **The zero-copy send threshold**, unmeasured, and whether the substrate
  should pick it at all.
- **Registration lifetime.** `IORING_REGISTER_BUFFERS` is not free and
  updating a registration has a cost that varies by kernel; a pool that
  grows by a slab must re-register or use a sparse registration, and
  which of the two is cheaper is unmeasured.
- **Multishot and backpressure.** A multishot receive appends to a
  socket's queue whether or not anyone is reading it, so a producer
  faster than its consumer grows that queue. The bound belongs here and
  is not designed.
