# Project workflow

How work is done in this repo. Not how the code is built
(ARCHITECTURE.md) nor what was decided (DECISIONS.md), but the order of
work that is the same for every task.

Copied from `limelight-lang/rfc` on 2026-08-12, because both repos have
the same owner and the same shape of work. Draft: Edmond has not yet
confirmed it for this repo.

## Branches

- Work commits **directly to `main`**; no PR is required.
- A side branch is optional for larger work: short kebab-case describing
  it.
- `main` is the mainline.

## Commits

- One line: `area: imperative summary`, lowercase area prefix and colon.
- `area` is the touched surface: `execution`, `switching`, `stacks`,
  `pool`, `reactor`, `cancel`, `deadlock`, `dev`.
- Body only when the *why* is not obvious; no diff retelling.
- English (core rule 17).

## PR and merge

- No PR required. One commit per logical change lands on `main`.

## Versions

- Not applicable yet — design phase, no releases.

## Language of documents

- English, including file names and headings.
- Measured figures carry the machine and the method, or they are marked
  as unmeasured. A comparative claim without a number does not go in.

## Test strategy

Written before the code so that the first module arrives with its checks
rather than acquiring them later. Nothing here has been run.

**Most of this substrate is single-threaded, and that is what makes it
testable.** A coroutine is touched only by its own thread, so the parking
protocol, the wake protocol, the wait record and the cancel path are
ordinary sequential code with an ordering that a test can drive step by
step. The concurrency is confined to a short list, and only that list
needs the expensive tools: the intake queue, the shared ready set, the
actor's readiness word, the mailbox, the cancelled byte, and the mounting
compare-exchange of a declared actor.

**Four kinds of check, and each has a place.**

- **Ordinary tests**, per module, for everything sequential. The
  protocols are the important ones: a test drives park and wake step by
  step and asserts what each step left behind.
- **Permutation tests** over the concurrent list above, with a model
  checker for the memory model rather than a stress loop. A stress loop
  that passes proves nothing about an ordering it did not happen to
  produce.
- **Miri** over every module that contains `unsafe`, which is most of
  them below `sched`.
- **A differential test per backend**: the same operation through
  `io_uring`, `IOCP`, `kqueue` and `epoll` has to produce the same
  observable result, and the readiness backends emulate completion, so
  the emulation is exactly where a difference hides.

**What checks each hot path of `ARCHITECTURE.md`:**

| Hot path | What checks it |
|---|---|
| the context switch | a round trip per platform row that asserts every callee-saved register, the control words and the TEB fields survived. Assembly, so nothing else can check it, and a missing register is silent corruption rather than a failure |
| park and wake | sequential protocol tests for steps 1 to 4 and 1 to 5; permutation tests for dispatch, the forward, and step 5's move path |
| completion dispatch | the differential test above, plus a test that a completion arriving at an empty waiter cell wakes nobody and still releases at owed-zero |
| a channel send and receive | sequential tests for the local kind; permutation tests for the shared kind, whose lock is a leaf and whose duty discharge has three carriers |
| enqueue into the ready set and ring a sleeper | **nothing checks the property that matters.** Correctness is a permutation test; the contention that made this a hot path is a measurement, and no benchmark exists |

**What nothing checks, by design or by absence.**

- **The foreign side of the C ABI.** A consumer's mutex keeping its owner
  field truthful, a consumer's channel holding its write ends in the
  registered handle table: the detector's promise never to invent a
  deadlock rests on both, and nothing verifies either. A resource that
  adopts neither is treated as always live, so the default costs
  detection rather than correctness.
- **A backend on a platform CI does not run.** Three targets gate a merge
  and run their tests — the two Linux ones and `x86_64-pc-windows-msvc` —
  and the rest are built and cross-checked only (`DECISIONS.md`,
  2026-08-13). So `io_uring`, `epoll` and `IOCP` are exercised and
  `kqueue` is compiled and never run, which makes the differential test
  above blind on exactly one backend.
- **Every performance claim.** `dev/BENCHMARKS.md` does not exist, and by
  the rule above a comparative claim without a number does not go in.
