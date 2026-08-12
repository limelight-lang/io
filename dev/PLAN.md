# PLAN

Updated: 2026-08-12 · Active: S1

A document closes when it answers its own question without reference to
the conversation it came from, and when every cross-reference in it
resolves.

## S1 — Repository skeleton  [in progress]

Goal: a fresh session can start from the repository instead of from the
conversation.
Done when: a session that has read only `dev/INDEX.md` names the reading
order and the step the work continues from.

- [x] S1.1 Local repository and the `dev/` journals
      done: INDEX, WORKFLOW, ARCHITECTURE, DECISIONS, POSTMORTEM and PLAN
        exist, and DECISIONS carries the decisions already taken
      tier: T2 · role: —
      handoff: repo at /home/edmond/limelight/io, branch main, one
        commit, no remote yet. WORKFLOW is a draft copied from
        limelight-lang/rfc and still needs Edmond's confirmation.
        Deadlock detection is deliberately not a decision: its mechanism
        is open and belongs to S4.1.
- [x] S1.2 Public `limelight-lang/io` and the first push
      done: `git ls-remote origin` returns refs
      tier: T1 · role: —
      handoff: https://github.com/limelight-lang/io, public, main pushed
        and tracking origin. The reactor is being split out; where the
        split lands (crate or repository) is not yet settled.

## S2 — Foundations  [ ]

Goal: the execution unit is specified before anything is built on it.
Done when: S3 and S4 can be written without reopening any question in S2.

- [ ] S2.1 `design/execution.md`
      done: the document states what a unit is, how the two coroutine
        kinds share one handle, and what happens at mount and unmount,
        including the TLS rule across a suspension point
      tier: T2 · role: —
- [ ] S2.2 `design/switching.md`
      done: the document states which registers a switch saves in each
        of the two cases, how the live-register mask reaches the switch,
        and how the foreign-frame bit is set and cleared
      tier: T2 · role: —
- [ ] S2.3 `design/stacks.md`
      done: the document states the reservation and commit scheme per
        platform, the size classes, the pooling and release protocol, and
        the mapping ceiling with the measured figures
      tier: T2 · role: —

## S3 — Objects and I/O  [ ]

Goal: an operation can be described end to end, from submission to the
release of its buffer.
Done when: the three buffer contracts are stated with their ownership
rules and their behaviour on each of the four backends.

- [ ] S3.1 `design/pool.md`
      done: the document states the object layout for coroutines, sockets
        and timers, the fields the wait edge occupies, and how a pool is
        walked while the system runs
      tier: T2 · role: —
- [ ] S3.2 `design/reactor.md`
      done: the document states the completion-first API, the three
        buffer contracts, how each degrades on IOCP, kqueue and epoll,
        and where zero-copy applies
      tier: T2 · role: —
- [ ] S3.3 `design/cancellation.md`
      done: the document states the two-phase teardown and the condition
        under which a stack and a buffer return to their pools
      tier: T2 · role: —

## S4 — Reliability  [ ]

Goal: a wait cycle is found while the process is busy, and reported as
data.
Done when: the detector is specified for both AND and OR waits, and its
boundary with the collector is stated.

- [ ] S4.1 `design/deadlock.md`
      done: the document picks between a maintained wait-for graph and a
        scan over the pools, and states both wait models, the validation
        of a result against a moving system, what is shared with the
        collector and what is not, and the victim policy
      tier: T2 · role: —
- [ ] S4.2 `README.md`
      done: a reader who has read only the README knows what the project
        is and in what order to read the design documents
      tier: T1 · role: —
