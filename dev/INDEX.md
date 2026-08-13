# dev/INDEX — project map for the agent

Pointers only. What is written elsewhere is not repeated here.

## What this repo is

Design documents for **io**: the execution and I/O substrate under
Limelight. It carries coroutines, the scheduler, the reactor, and the
deadlock detector. Limelight is its first consumer, not its only one:
the substrate is reachable over a C ABI from languages whose compilers
we do not own.

No product code yet. The design is being written first, stage by stage
([PLAN.md](PLAN.md)).

## Where to look

- **What the project is, and the reading order** → [../README.md](../README.md)
- **Plan and open steps** → [PLAN.md](PLAN.md)
- **Architecture / knowledge map** → [ARCHITECTURE.md](ARCHITECTURE.md)
- **Decisions (what, why, dated)** → [DECISIONS.md](DECISIONS.md)
- **Project conventions (branches, commits, PRs)** → [WORKFLOW.md](WORKFLOW.md)
- **Pitfalls already hit** → [POSTMORTEM.md](POSTMORTEM.md)
- **Diagrams** → [design/](design/) — PlantUML sources, no rendered images

## Words that changed meaning

**`half` is `entry`** (`dev/DECISIONS.md`, 2026-08-12): one entry of the
wait record per thing a coroutine waits on. Every design document uses the
new word; `DECISIONS.md` entries written before that date keep the old one,
because a superseded entry is never edited.

## Design documents

Written in the order below; each is listed here once it exists.

| Document | Answers |
|---|---|
| `design/execution.md` | what an execution unit is, how the two coroutine kinds share one reference and one wake path, how a unit is mounted on a thread |
| `design/switching.md` | narrow and full context switches, the live-register mask, the foreign-frame bit |
| `design/stacks.md` | reservation, lazy commit, pooling, size classes, the mapping ceiling |
| `design/pool.md` | slot layout for sockets, timers, operations and buffers; how the pools are walked; what is not a pool slot and where it lives instead |
| `design/reactor.md` | the completion-first API, the three buffer contracts, the four backends |
| `design/cancellation.md` | cancellation and two-phase teardown while the kernel still owns a buffer |
| `design/deadlock.md` | the liveness fixpoint inside the collector's walk, the three resource kinds, soft resolution by exception |
| `design/channels.md` | channels and futures: the two kinds and why they are two algorithms, the ends and the reachability constraint, closing and breaking, what a wake does not carry, the semaphore's release API |

## Related repositories

- `limelight-lang/rfc` — the Limelight design corpus. `runtime/actors.md`
  defines the actor contract this substrate mounts;
  `runtime/implementation-language.md` fixes Rust as the core language.
- `EdmondDantes/php-src`, `ext/async` — TrueAsync. Prior art and the
  source of the deadlock lesson recorded in `DECISIONS.md` (2026-08-12).

## Not yet present (deferred on purpose)

- `dev/BENCHMARKS.md` — no code to measure yet.
- `CHANGELOG.md` — nothing released.
- The ordering contract of the reactor's intake queue. `design/reactor.md`
  describes the queue without it, and `design/deadlock.md` rests on one
  queue per worker drained in order.
