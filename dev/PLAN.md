# PLAN

Updated: 2026-08-13 · Active: none — S6 waits on a decision

A document closes when it answers its own question without reference to
the conversation it came from, and when every cross-reference in it
resolves.

Stages S1 to S5 are finished and removed (rule 23.1.3). What survived
them: the decisions in [DECISIONS.md](DECISIONS.md), the two lessons in
[POSTMORTEM.md](POSTMORTEM.md), the eight design documents, the knowledge
map in [ARCHITECTURE.md](ARCHITECTURE.md), the three diagrams in
[design/](design/), and the test strategy in [WORKFLOW.md](WORKFLOW.md).
The numbers S1 to S5 are not reissued.

S5 closed without a Code Reviewer pass, which rule 23.1.3 asks for: that
role reads code style and this stage produced no code. Skipped for that
reason rather than by oversight.

## What is not yet planned

The first code needs a decision from Edmond, because the order is not
derivable from the documents: a vertical slice that reads a socket end to
end, or the substrate bottom-up starting at the pools and the switch. The
two differ in what they prove and in what they leave stubbed for months.
That stage is S6 and it does not start without the answer.

Open items carried by the documents rather than by this plan:

- each document's own "Open questions" section;
- `dev/BENCHMARKS.md` does not exist, and `design/switching.md` and
  `design/stacks.md` both name measurements that must be taken before
  their claims can be made;
- `rfc/BACKLOG.md` carries one defect raised here and owned there: mark
  termination waits on the slowest parked actor;
- `limelight-lang/model` owes the `walk` hook, its stage S18.
