# dev/INDEX — project map for the agent

Pointers only. What is written elsewhere is not repeated here.

## What this repo is

Design documents for **io-core**: the execution and I/O substrate under
Limelight. It carries coroutines, the scheduler, the reactor, and the
deadlock detector. Limelight is its first consumer, not its only one:
the substrate is reachable over a C ABI from languages whose compilers
we do not own.

No product code yet. The design is being written first, stage by stage
([PLAN.md](PLAN.md)).

## Where to look

- **Plan and open steps** → [PLAN.md](PLAN.md)
- **Architecture / knowledge map** → [ARCHITECTURE.md](ARCHITECTURE.md)
- **Decisions (what, why, dated)** → [DECISIONS.md](DECISIONS.md)
- **Project conventions (branches, commits, PRs)** → [WORKFLOW.md](WORKFLOW.md)
- **Pitfalls already hit** → [POSTMORTEM.md](POSTMORTEM.md)
- **Diagrams** → [design/](design/) — PlantUML sources, no rendered images

## Design documents

Written in the order below; each is listed here once it exists.

| Document | Answers |
|---|---|
| `design/execution.md` | what an execution unit is, how the two coroutine kinds share one handle, how a unit is mounted on a thread |
| `design/switching.md` | narrow and full context switches, the live-register mask, the foreign-frame bit |
| `design/stacks.md` | reservation, lazy commit, pooling, size classes, the mapping ceiling |
| `design/pool.md` | object layout for coroutines, sockets and timers; how the pools are walked |
| `design/reactor.md` | the completion-first API, the three buffer contracts, the four backends |
| `design/cancellation.md` | cancellation and two-phase teardown while the kernel still owns a buffer |
| `design/deadlock.md` | how a wait cycle is found, AND and OR waits, victim selection |

## Related repositories

- `limelight-lang/rfc` — the Limelight design corpus. `runtime/actors.md`
  defines the actor contract this substrate mounts;
  `runtime/implementation-language.md` fixes Rust as the core language.
- `EdmondDantes/php-src`, `ext/async` — TrueAsync. Prior art and the
  source of the deadlock lesson recorded in `DECISIONS.md` (2026-08-12).

## Not yet present (deferred on purpose)

- `README.md` — written last, because it states the reading order over
  finished documents (PLAN.md, S4.2).
- `dev/BENCHMARKS.md` — no code to measure yet.
- `CHANGELOG.md` — nothing released.
