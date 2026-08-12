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

## Corrections of 2026-08-12, not yet applied to every document

The design documents were written before the coroutine's shape was
settled against `limelight-lang/model`. Four things in them are retired,
and `dev/DECISIONS.md` of that date is what holds where they disagree:

| Retired | What holds instead | Still stale in |
|---|---|---|
| a 64-bit handle carrying a pool slot and a generation | a counted reference to the coroutine object | `design/pool.md` |
| slots in a bespoke array, with deferred reclamation | the coroutine is an entity in the memory manager; pools remain for sockets, timers, operations and buffers | `design/pool.md` |
| a compare-and-swap on every parking transition, and the `Parking` state | plain stores: a unit is touched only by its own thread | `design/cancellation.md` |
| a seqlock over the wait record, and generation checks on the wake path | the epoch alone, for telling this wait from the previous one | `design/pool.md`, `README.md` |
| the detector's claim word, and a search rooted at one suspect | one detector on the collector's thread, one fixpoint over the whole heap | `design/pool.md` |
| a `poster` field in the wait record, and occupant validation on the wake path | the entry names a resource; the resource answers who can end the wait | `design/pool.md` |

`design/execution.md`, `design/switching.md`, `design/stacks.md` and
`design/deadlock.md` are already corrected. Applying the rest belongs to
the architecture stage.

One question has to be settled while applying them, because three
documents give two answers: **when the wait epoch moves.**
`design/cancellation.md` bumps it when a wait ends, `design/pool.md` moves
it only at the next park, and the wake protocol in `design/execution.md`
has no bump step at all. The detector's confirmation on the owner thread
(`design/deadlock.md`, step 12) compares the epoch, so the answer is an
input to its correctness and not a detail of wording.

**`half` is renamed `entry`** (`dev/DECISIONS.md`, 2026-08-12): one entry
of the wait record per thing a coroutine waits on. Renamed in
`design/execution.md`, `cancellation.md`, `pool.md`, `reactor.md` and
`deadlock.md`. `README.md` still says half. Earlier `DECISIONS.md` entries
keep the old word, because an entry there is superseded and never edited.

## Design documents

Written in the order below; each is listed here once it exists.

| Document | Answers |
|---|---|
| `design/execution.md` | what an execution unit is, how the two coroutine kinds share one handle, how a unit is mounted on a thread |
| `design/switching.md` | narrow and full context switches, the live-register mask, the foreign-frame bit |
| `design/stacks.md` | reservation, lazy commit, pooling, size classes, the mapping ceiling |
| `design/pool.md` | slot layout for sockets, timers, operations and buffers; how the pools are walked; what a resource answers about a wait |
| `design/reactor.md` | the completion-first API, the three buffer contracts, the four backends |
| `design/cancellation.md` | cancellation and two-phase teardown while the kernel still owns a buffer |
| `design/deadlock.md` | the liveness fixpoint inside the collector's walk, the three resource kinds, soft resolution by exception |

## Related repositories

- `limelight-lang/rfc` — the Limelight design corpus. `runtime/actors.md`
  defines the actor contract this substrate mounts;
  `runtime/implementation-language.md` fixes Rust as the core language.
- `EdmondDantes/php-src`, `ext/async` — TrueAsync. Prior art and the
  source of the deadlock lesson recorded in `DECISIONS.md` (2026-08-12).

## Not yet present (deferred on purpose)

- `dev/BENCHMARKS.md` — no code to measure yet.
- `CHANGELOG.md` — nothing released.
- A document for channels and futures. `design/deadlock.md` depends on
  their semantics — dropping the last write end closes a channel and
  breaks a future, and no traced reference runs from the resource back to
  its ends — and nothing defines them.
- The ordering contract of the reactor's intake queue. `design/reactor.md`
  describes the queue without it, and `design/deadlock.md` rests on one
  queue per worker drained in order.
