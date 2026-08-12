# PLAN

Updated: 2026-08-12 · Active: S3

A document closes when it answers its own question without reference to
the conversation it came from, and when every cross-reference in it
resolves.

## S1 — Repository skeleton  [done]

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

## S2 — Foundations  [done]

Goal: the execution unit is specified before anything is built on it.
Done when: S3 and S4 can be written without reopening any question in S2.

- [x] S2.1 `design/execution.md`
      done: the document states what a unit is, how the two coroutine
        kinds share one handle, and what happens at mount and unmount,
        including the TLS rule across a suspension point
      tier: T2 · role: Critic
      Critic 2026-08-12 round 1, on the section outline: twelve findings,
        six of them high. The outline funnelled nothing — a stackful park
        (a switch that never returns) and a stackless park (a record write
        and `Pending` up the poll chain) reached the park primitive by
        different paths, leaving the detector a hole it cannot report. It
        blessed "parked" as a safepoint, though an I/O park suspends
        mid-message while the actor contract rests on an empty stack
        between messages. It gave wake no section, so the race between a
        completion on another thread and an unfinished suspension had
        nowhere to be resolved. It shaped the wait record for one wait,
        though await-with-timeout is an OR of two. It stated a TLS rule
        that foreign frames break by construction (errno, openssl's error
        queue). And it left the poll ABI the Limelight compiler owes the
        substrate unnamed, which `pool.md` would have had to invent.
        Accepted in full; outline rewritten from ten sections to thirteen.
      Critic 2026-08-12 round 2, on the written text: eighteen findings,
        five high, and the parking protocol was wrong as written. A waker
        finding `Running` had no legal move, so a reply that arrived
        before the suspension was lost; the fix orders the protocol —
        state first, record second, arm third — which makes `Running`
        unreachable for a waker. `Parked` was published by the unit, which
        announces a context still being saved; the worker publishes it
        now, after the switch. AND waits could not terminate, because no
        waker was allowed to write the record; wakers now decrement a
        counter. A retired loser of an OR wait could wake the unit out of
        an unrelated wait; every waker validates the epoch first. Results
        had no channel at all. Accepted in full; the protocol sections
        were rewritten and "Waking" added as a section of its own.
      handoff: design/execution.md, 15 sections. The parking protocol is
        the load-bearing part: four states, every transition a CAS, the
        order state→record→arm, and `Parked` published by the worker.
        Exactly one enqueue per wake, decided by that CAS. Open and owed
        to another repo: mark termination waits on the slowest parked
        operation, which belongs in rfc/BACKLOG.md and is not there.
- [x] S2.2 `design/switching.md`
      done: the document states which registers a switch saves in each
        of the two cases, how the live-register mask reaches the switch,
        and how the foreign-frame bit is set and cleared
      tier: T2 · role: Critic
      Critic 2026-08-12 round 1: the narrow switch was unsound as
        designed. A per-site live-register mask cannot see the frames
        above it, so a callee-saved register holding an ancestor's value
        goes unsaved and another unit overwrites it — silent corruption,
        no crash. Correcting the mask to "live, plus every callee-saved
        register this frame did not spill" restores nearly the full set
        and erases the gain, so the mask was dropped: narrowing is now a
        calling convention with no callee-saved registers along
        suspendable paths, and the park primitive's tail is assembly
        because a rustc-compiled frame between the site and the switch
        would carry live values of its own. Four more findings held. The
        foreign-frame marker is a counter, not a bit: foreign code calls
        back into ours, and the inner return would clear a marker the
        outer frame still needs, which also mis-answers migration and
        force-killability. corosensei carries no CET sequence — verified
        in its x86_64 source — so "the library provides the full switch"
        was false on any host with user shadow stacks. The TEB swap
        omitted GuaranteedStackBytes, and "read the offset from the
        running TEB" was incoherent. The QEMU assembly-coroutine citation
        was wrong: that backend was never merged. Accepted in full;
        document rewritten.
      handoff: design/switching.md, 200 lines. The load-bearing reversal
        is that narrowing is a calling convention, not a mask, and the
        park primitive's tail must be assembly for it to apply. The
        foreign-frame marker is a counter with three readers — the
        switch, migration, force-killability — so a unit that opts out of
        maintaining it reads as permanently non-zero. CET is ours to
        write: corosensei has no shadow-stack support.
- [x] S2.3 `design/stacks.md`
      done: the document states the reservation and commit scheme per
        platform, the size classes, the pooling and release protocol, and
        the mapping ceiling with the measured figures
      tier: T2 · role: Critic
      Critic 2026-08-12 round 1: twenty-two findings, seven high, and one
        of them removed half the document. The release gate — "a stack
        returns only after the kernel holds no reference into it" — was
        unevaluable, because the wait record is overwritten on every park
        and nothing survived a unit to remember an operation still in
        flight; worse, the gate protected pool reuse but not the same
        running unit laying new frames over a retired read's buffer. The
        answer was to remove the case: nothing the kernel touches lives
        on a stack, so a stack's lifetime is exactly its unit's. Six more
        held. Overflow-by-unwinding is not implementable as promised —
        ARM EHABI has no instruction-precise unwinding, Rust's panic path
        is not async-signal-safe, and Windows uses SEH — so the default
        is now process death with a per-platform opt-in. `MEM_RESET` on
        Windows releases nothing: the commit charge stays, which is a
        leak on the platform where the table claimed release.
        `MAP_NORESERVE` is ignored under `vm.overcommit_memory=2`.
        GCC assumes a 64 KB guard on AArch64, so a one-page band is
        stepped over by probed code. Every figure was a 4 KB-page figure
        on targets that run 16 KB pages. Release ordering was
        unspecified, and one order corrupts. Accepted in full; document
        rewritten.

## S3 — Objects and I/O  [in progress]

Goal: an operation can be described end to end, from submission to the
release of its buffer.
Done when: the three buffer contracts are stated with their ownership
rules and their behaviour on each of the four backends.

- [ ] S3.1 `design/pool.md`
      done: the document states the object layout for coroutines, sockets
        and timers, the fields the wait edge occupies, and how a pool is
        walked while the system runs
      tier: T2 · role: Critic
- [ ] S3.2 `design/reactor.md`
      done: the document states the completion-first API, the three
        buffer contracts, how each degrades on IOCP, kqueue and epoll,
        and where zero-copy applies
      tier: T2 · role: Critic
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
