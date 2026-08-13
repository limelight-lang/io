# PLAN

Updated: 2026-08-12 · Active: S5

A document closes when it answers its own question without reference to
the conversation it came from, and when every cross-reference in it
resolves.

Stages S1 to S4 are finished and removed (rule 23.1.3). What survived
them: the decisions in [DECISIONS.md](DECISIONS.md), the lesson in
[POSTMORTEM.md](POSTMORTEM.md), and the seven design documents themselves.
The numbers S1 to S4 are not reissued.

## S5 — Implementation architecture in dev/  [in progress]

Goal: the stage after this one writes code instead of settling boundaries.
Done when: no module's shape is decided in conversation only, and no
design document describes machinery that has been retired.

- [x] S5.1 The deadlock detector, settled and written down
      done: `design/deadlock.md` states one algorithm with no retired
        machinery in it, and `dev/DECISIONS.md` carries what was chosen
      tier: T2 · role: Sage
      Sage 2026-08-12 round 1: liveness must be computed per entry, not
        per coroutine — a coroutine whose AND wait holds a kernel entry
        beside a dead mutex is marked live and a real cycle is lost
        forever. Accepted; the arming rule in `design/deadlock.md` carried
        the same defect and was corrected with it.
      Sage 2026-08-12 round 2: the detector marks no resource. A channel
        closes when its last write end is dropped, which the resumed
        coroutine does while unwinding, so co-waiters fail without a
        second pass; a detector-placed mark would turn a resurrected
        writer's legitimate write into an error the detector invented.
        Accepted. Also refused to relax the validation now that resolution
        is an exception rather than a kill: a spurious close is a
        semantic lie no catch block repairs. Accepted.
      Sage 2026-08-12 round 3, on the written documents: the arming rule
        had lost the OR clause, so a knot of three coroutines armed no
        watchdog at all; `design/execution.md` still drew the `Parking`
        state and allocated units from a pool while `dev/INDEX.md`
        declared it corrected; the debtor-only edge model survived in
        `execution.md` and `pool.md`, instructing a sender registry the
        detector never reads. All accepted and fixed, together with six
        smaller findings; the two that were not defects of mine —
        `pool.md` and `cancellation.md` carrying retired machinery no
        correction row named — went into the table in `dev/INDEX.md`.
      Sage 2026-08-12 round 4, on the fixes: three of them were wrong.
        The new liveness root for a resource with a pending deposit was
        read by no rule, so the false report it was meant to prevent
        survived; the rewritten cancel paragraph enumerated its stores and
        left out the `Woken` store and the enqueue, which is a unit that
        never resumes; `design/pool.md` came back with a two-kind
        taxonomy against the document's three. All fixed. Its remaining
        seven, including the object-model constraint a channel must obey
        for reachability to mean anything, are folded into S5.2 and into
        the table in `dev/INDEX.md`.
      Sage 2026-08-12 round 5: the served set covered deposits only while
        the liveness table also consulted it for a pending take, so a
        sender parked on a full channel was still reported dead; the
        cancel path's internal order contradicted the wake protocol it
        shares a code path with; the semaphore existed in `pool.md` alone,
        with no liveness rule and a single-owner field that a counting
        semaphore does not have. Fixed, the semaphore by giving it every
        current permit holder and a rule that any one of them releasing is
        enough — my decision, not Sage's, and the next round judges it.
      Sage 2026-08-12 round 6: that semaphore rule rejected, and rightly.
        A release needs no prior acquire, so a semaphore created with zero
        permits has no holders at all, and the rule would have failed the
        waiter of a live producer within one threshold. Replaced by its
        ruling: a semaphore released only through a permit guard stays a
        debtor with permit-counting liveness, one posted by anyone becomes
        a supply resource under handle reachability, and everything over
        the C ABI is postable because its discipline is invisible to us.
        The served set's list of forms is now examples rather than a closed
        enumeration, for the same reason.
      handoff: the algorithm is a least fixpoint over the collector's
        walk, two mark bits, three resource kinds, one victim per sink
        SCC, owner-thread confirmation before anything is written.
        `half` is renamed `entry` across five design documents.
- [x] S5.8 The actor run queue and its stealing, researched and decided
      done: `dev/DECISIONS.md` carries an entry naming the chosen
        structure, every candidate it beat and the property it beat them
        on, and what stays unmeasured until there is code
      tier: T2 · role: Critic
      Edmond 2026-08-13: `crossbeam-deque` was taken, never compared, so
        it is not final. A pinned coroutine's queue is a classic ring
        buffer and is not part of this question — only the actor queue is.
      Runs before S5.2 because `dev/ARCHITECTURE.md` (S5.4) names the
        scheduler's types and cannot be written around an open dependency.
      Critic 2026-08-13 round 1: nine high findings, four of them
        structural — an idle actor has no unit, so nothing arbitrated its
        placement and neither the one-enqueue invariant nor the foreign-frame
        counter applied; the movability vet was available only where the
        decision was already local; a shed lane per owner restored the O(N)
        foreign scan that stealing was rejected for; and filling a lane at a
        turn boundary tied the offload rate to the busiest worker's boundary
        rate. Accepted; the entry was rewritten around one shared ready set.
      Critic 2026-08-13 round 2: five critical. The mounting transitions of
        the actor's readiness word were missing, so an actor could run on two
        workers at once; forwarding a wake to a unit that moved mid-message
        races on the wait record and terminates only under a claim, which
        costs an atomic per wake; the handshake answer treated the
        woken-mid-message list as consistent, which `design/execution.md`
        denies; nothing rings a sleeping worker when an actor enters the
        ready set; and the actor-call liveness rule of `design/deadlock.md`
        reads no field that is true between taking a message and mounting
        it, so a healthy synchronous call is reported deadlocked. Fixes in
        hand; the price of a mid-message move goes back to Edmond.
      Edmond 2026-08-13: pay the price only for actors that declare
        mid-message movement, and at most one move per message.
      Sage 2026-08-13: the claim is not a lock a forward holds — that would
        block mounting on another thread's progress. It is the unit's state
        word widened for a declared actor to `Running(W)`, `Parked(W)`,
        `WokenLocal(W)`, `WokenShared`, `Terminal`, with the cancelled bit
        moved to a byte beside it; any thread holding a signal reads the word
        once and either applies it, forwards it to the named worker, or drops
        it. Termination comes from Edmond's one-move limit, not from the
        claim: the owner chain within one message is at most two long. No hole
        in the limit, because a move happens only through a wake and a wake
        only on a decided wait, so a moved unit has no outstanding entries.
        Root the memory mark in the scheduler's ownership table and in no
        queue — the table is a lifetime registry a move never touches — under
        four queue invariants, which also repairs a defect of mine: the entry
        had named every private list a root, and those are non-atomic
        single-owner lists no foreign thread may read. Also refused the
        Critic's two alternatives: no second inbound queue, and no ban on
        moving with foreign-armed entries. Final; executed as written, with
        `design/execution.md` owing the statement that `Woken` means the wait
        is decided, and actor creation owing the rejection of a declaration
        paired with opting out of the counter.
      handoff: three entries in `dev/DECISIONS.md` of 2026-08-13 carry it —
        re-mounting and its price with the claim protocol, the wait epoch, and
        the scheduling structure. `design/execution.md` gained dispatch as
        step 0 of `wake`, the split step 5, the widened word for a declaring
        actor, the mounting compare-exchange and the `Woken` equivalence;
        `design/deadlock.md`'s actor-call row now reads the readiness word.
        `crossbeam-deque` is out of the toolchain and `dev/DECISIONS.md` of
        2026-08-12 keeps it only as the superseded choice.
- [x] S5.2 Channels and futures get a document that owns their semantics
      done: closing on the drop of the last write end is defined there,
        the ends reference the resource while the resource holds no traced
        reference back to them, and `design/deadlock.md` cites both
        instead of assuming them
      tier: T2 · role: Critic → Sage
      Edmond 2026-08-13: two kinds of channel as in TrueAsync, because a
        channel across threads is a different algorithm.
      Critic 2026-08-13 round 1: eighteen findings, all accepted. The two that
        were corpus-level: a write end in a stackful frame is reachable by
        nothing, since the `walk` hook covers a coroutine's cells and no
        document enumerates its stack, so a healthy receiver is failed while
        its producer merely holds the end in a local; and the channel's waiter
        queue is a traced edge into a parked coroutine, so the liveness mark
        flows in and no channel waiter ever enters the dead set, with scope
        binding making every channel L-reachable. Also: no ordering rule
        between arming and depositing, a refused wake absorbed, a non-blocking
        rendezvous send with nowhere to put its value, arm-time satisfaction
        breaking the FIFO contract and resetting timeouts, a thread-local end
        in a migrating actor's hands, a miscitation of `rfc/runtime/actors.md`
        that admitted an arena-born value into a shared channel.
      Critic 2026-08-13 round 2: seventeen more, five critical. The node of an
        entry satisfied inline was never unlinked, which poisons the queue on
        every fast-path receive; retiring "everything armed earlier" is wrong
        under AND and strands the wait; a cancel found in `Running` and every
        park under final cancellation leave nodes armed into teardown; an
        *accepted* wake is swallowed by a waiter cancelled before it consumes;
        and the one lock is either held across `wake`, whose retirement takes
        other channels' locks and deadlocks two workers ABBA, or the walk is
        not a walk. Plus two of mine: `deadlock` as a fifth close cause is the
        resource mark `deadlock.md` forbids, and the end count reaching zero
        before the closed flag invents a deadlock on an ordinary producer exit.
      Sage 2026-08-13: the queue link stays counted and traced — an untraced
        counted reference is unsound in this memory model — and the isolation
        is a flow rule instead: L does not flow through a supply resource's
        waiter links, and the attribution walk does not cross them either, the
        second half being mine to have missed, since without it a dead
        coroutine reaches other write ends through the queue and the victim is
        chosen from a distorted graph. The walk-until-accepted rule is replaced
        by conservation of duty: a supply wake is discharged only by
        consumption, and each of the three carriers — inline refusal, a
        refusal learned on another worker, an accepted wake never consumed —
        names who re-runs it, which is why a supply wake's intake payload now
        carries the resource. The channel's lock is a leaf: nothing foreign
        runs under it, `wake` never does, and "link then test" is replaced by
        one decide-and-publish section. A detector verdict does not close a
        channel; the cause lives in the error. The count-versus-flag race is
        closed by reading closedness as "flag or end count at zero" and by
        extending the validation and the owner-side check to the resource's
        own fields. Final; executed as written.
      handoff: `design/channels.md` is the third version and the one that
        stands. It changed `design/deadlock.md` (the supply constraint's second
        half, the L and attribution flow rule, closedness, steps 8 and 12, and
        a new open item for the mailbox row that no document owns),
        `design/execution.md` (inline satisfaction and AND, the three-valued
        `wake`, the cancelled bit at park step 4) and `design/cancellation.md`
        (the channel row, with duty discharge). Two holes stay open by name: a
        write end the tracer cannot reach, and the intake queue's ordering
        contract.
- [x] S5.3 Apply the remaining corrections listed in `dev/INDEX.md`
      done: the corrections table in `dev/INDEX.md` is empty and deleted,
        and the wait epoch has one rule instead of the three documents'
        two
      tier: T2 · role: Critic → Sage
      Critic 2026-08-13 round 1, two fans. Correctness: the cancel path
        claimed a wait with nothing to claim it in, since the winner field
        existed under OR alone, so a retired entry firing later drove an AND
        counter to zero and enqueued the unit twice; `Terminal` had no place
        in teardown, and a terminated coroutine was an L root, which made its
        waiter live forever. Completeness: 25 findings, mostly stale
        citations — work stealing, forced teardown, the unit slot, the
        detector reading the pools. All accepted; the six against
        `dev/ARCHITECTURE.md` were left to S5.4, which rewrites that file.
      Sage 2026-08-13 round 1: a slot names its waiter through a waiter cell
        with one writer, one emptier and one thread, emptied by the
        completion or by the retire; a multishot names the socket and never a
        unit; the cells are memory roots. A cross-thread cancel carries a
        promise the applying worker resolves, and a cancel in `WokenShared`
        is the cancelled byte. Final.
      Critic 2026-08-13 round 2: thirteen more against the fixes, one
        critical. The cell root was a slab walk licensed to miss slots; the
        counted reference in a debtor's owner field conducted L, so no mutex
        cycle was reportable at all; the generation-wrap argument had become
        circular; the socket queue had no liveness row; the multishot recheck
        ran before the publish it defended; a cancel had nowhere to store its
        error but an entry, which the next wake overwrote.
      Sage 2026-08-13 round 2, two calls, both Final. Protocol: the record
        gains its own result slot and a reserved winner value; a
        `WokenShared` cancel writes the byte, re-reads the word and answers
        from the re-read; the socket queue reads always live and the reactor
        reports re-arm starvation; the recheck belongs to whoever writes the
        cell. Memory: struck its own root sentence — the ownership table
        covers an undecided wait and an intake entry covers the one
        cross-thread retire, so no pool is a root; L stops at a debtor's
        owner field; 32 bits of generation stand on the quiescence bound,
        which makes "no pool handle crosses the API" load-bearing.
      handoff: the corrections table and the epoch disagreement are gone from
        `dev/INDEX.md`, whose corrections section is deleted; `design/pool.md`
        no longer describes channels, semaphores or units, and owns the waiter
        cell instead. Three `dev/DECISIONS.md` entries of 2026-08-13 carry the
        rulings, the third striking a sentence of the first. Nothing is
        committed: the working tree holds all of it.
- [x] S5.4 `dev/ARCHITECTURE.md` from provisional to real
      done: every module row names its files, its public types and what
        crosses its boundary; no row rests on retired machinery
      tier: T2 · role: Critic → Sage
      Critic 2026-08-13 round 1: 32 findings, five critical. The intake
        queue was given to `io-reactor` and then drained, written and read
        by three `io-core` modules; `unit` held `park` and `wake`, whose
        steps name the scheduler, the channels and the reactor; the
        detector's row denied depending on `pool` while naming pool code;
        "everything crossing a thread boundary" was refuted by the same
        file three lines down. Timer wheel, diagnostic dump, C-ABI handle
        table, actor header, `io-api` and the file-I/O thread pool had no
        owner at all.
      Sage 2026-08-13: the intake queue is `sched`'s, whole, in `io-core`,
        an entry being a declaration and an applier; what crosses the crate
        line instead is the worker driver. `park` and `wake` are `sched`'s
        exports and `unit` keeps the record mechanics, split by step. `sync`
        is a module of `io-core` and supersedes the 2026-08-12 module list.
        Final.
      Critic 2026-08-13 round 2: 22 more, two critical, and both hit the
        same device — the strict level table, which by then had been wrong
        three times. The detector roots its liveness mark in a table
        `io-api` owns; the watchdog's arming rule runs on the park path.
        Rather than defend it a third time the levels were replaced by a
        call graph with two other kinds of edge named beside it: an
        installed callable and a read of a root.
      handoff: `dev/ARCHITECTURE.md` is nine modules and the API crate, a
        section each, and the ruling behind its boundaries is in
        `dev/DECISIONS.md` of 2026-08-13. Four opaque callables are the only
        things that cross a level or a crate line: the cancel handle, the
        worker driver, `sync`'s duty hook, the detector's arming predicate.
        Three modules have no design document and `dev/INDEX.md` now says
        which — the scheduler, the synchronisation primitives, the timer
        wheel — plus the whole C-ABI surface.
- [ ] S5.5 PlantUML in `dev/design/`: parking, submission and completion,
      the object lifecycle
      done: the parking diagram matches the numbered protocol in
        `design/execution.md` step for step
      tier: T2 · role: Critic
- [x] S5.6 Toolchain and build, as a `dev/DECISIONS.md` entry
      done: the rustc version and where it comes from (the single-LLVM
        rule in the rfc), edition, MSRV, backend features, CI targets
      tier: T1 · role: —
      Taken out of order, while S5.4 waited on a Sage ruling: it depends on
        the rfc and on `design/reactor.md` and `design/switching.md`, and on
        nothing S5.4 or S5.5 decides.
      handoff: `dev/DECISIONS.md` of 2026-08-13 carries it. The toolchain is
        pinned in `rust-toolchain.toml` rather than bounded by an MSRV,
        because the C++ layer must hold rustc's own LLVM; pinned today is
        rustc 1.96.0 with LLVM 22.1.2, edition 2024, all read by running the
        compiler. Two version checks are owed to CI, and both fail on this
        machine unpinned: system LLVM is 22.1.8 and system Clang is 18.1.3.
        Seven CI targets follow the platform table of `design/switching.md`;
        which of them ever runs its tests is open.
- [ ] S5.7 Test strategy, as a section of `dev/WORKFLOW.md`
      done: every hot path named in `dev/ARCHITECTURE.md` says what checks
        it, or says that nothing does
      tier: T1 · role: —

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
