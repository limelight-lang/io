# io

**io** is the execution and I/O substrate under
[Limelight](https://github.com/limelight-lang/rfc): coroutines and the
scheduler that mounts them, a completion-first reactor, and a deadlock
detector that works while the process is busy. It is written in Rust and
reachable over a C ABI, so a language whose compiler we do not own can
use it without a runtime of its own.

> **Status**: design phase. These documents fix the contracts before the
> first line of code. Open steps are in [dev/PLAN.md](dev/PLAN.md).

## The shape

| Pillar | Idea | Document |
|---|---|---|
| Two coroutine kinds | Stackful by default, because only a stackful coroutine can suspend below a frame someone else compiled. Stackless where we own the compiler and the body never crosses a foreign ABI. Both behind one counted reference and one wake path | [execution](design/execution.md) |
| Parking as a protocol | Three states, plain stores, and an order — record, then arm, then suspend — that guarantees a completion finds somewhere to record itself. A coroutine is touched only by the thread its state word names — for all but a declared actor, the one it has always run on — so the wake that once arrived mid-suspension cannot | [execution](design/execution.md) |
| Switching by convention | A live-register mask cannot narrow a switch soundly, because callee-saved registers hold ancestors' values. Narrowing is a calling convention with no callee-saved registers along suspendable paths | [switching](design/switching.md) |
| Stacks reserved, committed lazily | 2 MB of address space costs 4 KB of memory at one page touched, measured. No growth, no segments: the ceiling is kernel mappings, not memory | [stacks](design/stacks.md) |
| Pools for what the kernel touches | Sockets, timers, buffers and operations live in walkable slabs: enumeration needs no registry, and a buffer pool can be registered with the kernel once. A coroutine is not among them — it is an object of the memory manager, and its lifetime is its reference count | [pool](design/pool.md) |
| Completion-first I/O | The API says "do this with this memory and wake me", never "tell me when it is ready". io_uring and IOCP are native; kqueue and epoll emulate. Three buffer contracts, because the kernel's ownership of a buffer is the contract | [reactor](design/reactor.md) |
| Cancellation in two phases | Cancelling asks the kernel to finish sooner; it does not take the buffer back. Release happens on the original completion, never on the cancel's | [cancellation](design/cancellation.md) |
| Deadlock as data | Detection triggers on wait age, not on the process falling quiet, so a cycle among three coroutines is found while two hundred others work. No graph is built: liveness is a fixpoint over the collector's own walk, and a proved-dead wait fails with an exception at its wait point | [deadlock](design/deadlock.md) |

## Two rules that carry the rest

**Nothing the kernel touches lives on a coroutine's stack.** Buffers, `iovec`
arrays and submission structures come from the buffer pool. A
completion-based kernel owns that memory past the call that submitted it,
so a stack-resident buffer would outlive its frame, and a stack's
lifetime would stop being its coroutine's.

**Every suspension goes through one park primitive.** A wait edge has two
ends: the coroutine's end names the resource it waits on, and the resource's end
answers who can end that wait. Recording only the first end is what makes a
deadlock undetectable, and it is the mistake this design exists to avoid.

## Reading order

1. [design/execution.md](design/execution.md) — the coroutine, the two kinds,
   parking and waking. Everything else refers to it.
2. [design/pool.md](design/pool.md) — where objects live and how they are
   walked.
3. [design/reactor.md](design/reactor.md) — how an operation is submitted
   and completed.
4. [design/cancellation.md](design/cancellation.md) — how it ends early.
5. [design/deadlock.md](design/deadlock.md) — what the records add up to.

[design/switching.md](design/switching.md) and
[design/stacks.md](design/stacks.md) sit under the first and can be read
when the machine details matter.

## Crates

| Crate | Holds |
|---|---|
| `io-core` | the coroutine, switching, stacks, the scheduler, the pools, the detector |
| `io-reactor` | submission and completion, with the four backends behind cargo features |
| `io-api` | the public surface and the C ABI |

Dependencies point from `io-api` down to `io-reactor` down to `io-core`.
The reverse edge does not exist, so a consumer running computation
without I/O compiles none of the platform backends.

## For the agent

[dev/INDEX.md](dev/INDEX.md) is the map: decisions in
[dev/DECISIONS.md](dev/DECISIONS.md), conventions in
[dev/WORKFLOW.md](dev/WORKFLOW.md), the knowledge map in
[dev/ARCHITECTURE.md](dev/ARCHITECTURE.md), open steps in
[dev/PLAN.md](dev/PLAN.md).
