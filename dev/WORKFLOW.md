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
