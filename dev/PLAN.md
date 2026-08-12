# PLAN

Updated: 2026-08-12 · Active: none

A document closes when it answers its own question without reference to
the conversation it came from, and when every cross-reference in it
resolves.

Stages S1 to S4 are finished and removed (rule 23.1.3). What survived
them: the decisions in [DECISIONS.md](DECISIONS.md), the lesson in
[POSTMORTEM.md](POSTMORTEM.md), and the seven design documents themselves.
The numbers S1 to S4 are not reissued.

## What is not yet planned

The design corpus is complete and no code exists. A stage for the first
code needs a decision from Edmond, because the order is not derivable from
the documents: a vertical slice that reads a socket end to end, or the
substrate bottom-up starting at the pools and the switch. The two differ
in what they prove and in what they leave stubbed for months.

Open items carried by the documents rather than by this plan:

- each document's own "Open questions" section;
- `dev/BENCHMARKS.md` does not exist, and `design/switching.md` and
  `design/stacks.md` both name measurements that must be taken before
  their claims can be made;
- `rfc/BACKLOG.md` carries one defect raised here and owned there: mark
  termination waits on the slowest parked actor.
