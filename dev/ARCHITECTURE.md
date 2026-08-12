# Architecture — knowledge map

Who knows what, and what each module must not know. Written before the
code, and rewritten whenever a boundary moves rather than after.

Provisional: the modules below are the ones the design documents are
being written against (`PLAN.md`). A row hardens when its document
closes.

## Crates

One repository, one cargo workspace, three crates. The split follows the
dependency direction, so a consumer that runs computation without I/O
compiles none of the platform backends.

| Crate | Holds | Depends on |
|---|---|---|
| `io-core` | `unit`, `switch`, `stack`, `sched`, `pool`, `deadlock` | memory manager |
| `io-reactor` | `reactor` and the four backends, each behind a cargo feature | `io-core` |
| `io-api` | the public surface and the C ABI | `io-core`, `io-reactor` |

`io-reactor` knows `io-core`; the reverse dependency does not exist and
must not appear. Cancellation crosses this boundary, so it is split
rather than shared: `io-core` cancels a unit through an opaque cancel
handle stored in the wait record, and whoever parked the unit installed
that handle. `io-reactor` supplies one that submits the kernel-side
cancel and waits for its completion.

## Modules

| Module | Responsible for | Knows | Does not know | Depends on |
|---|---|---|---|---|
| `unit` | the execution unit and its handle | the two suspension kinds, the discriminant bit, the wait record | which backend completed an operation, what a socket is | `stack` |
| `switch` | context switches | register sets per ABI, the TEB and shadow-stack swaps, the foreign-frame counter, the suspendable calling convention | scheduling policy, I/O, actors, the unit slot layout | `stack` |
| `stack` | stack memory | reservation, lazy commit, size classes, pooling, guard bands | what runs on a stack | memory manager |
| `sched` | mounting units on threads | run queues, stealing, mount and unmount hooks, actor context install | how a unit suspends internally, how an operation completes | `unit`, `switch` |
| `pool` | object storage and enumeration | the layout of coroutines, sockets and timers; how each pool is walked | the meaning of a wait edge, the reactor backend | memory manager |
| `reactor` | submitting operations and delivering completions | the four backends, the three buffer contracts, buffer ownership | scheduling policy, the wait-for graph | `pool`, `sched` |
| `cancel` | cancelling in-flight work; split across the two crates | unit side: the cancel handle and the teardown order. Operation side: what the kernel still owns and when it lets go | unit side: which backend owns the operation. Operation side: why the unit is being cancelled | `unit`, `reactor` |
| `deadlock` | finding wait cycles | how parked units are enumerated, AND and OR waits, how a result is validated against a moving system | how an operation is submitted, how a stack is allocated | `pool`, `unit` |

## Layering

Dependencies point down and there are no cycles:

```
deadlock   cancel
      \      |
       \  reactor
        \   /
         pool        sched
            \        /   \
             \      /     switch
              unit          |
                 \          |
                  \       stack
                   \      /
                 memory manager
```

`deadlock` reads what `unit` and `pool` already store, and writes
nothing. `cancel` is the only module allowed to tell `reactor` that an
operation must end early.

## Shared resources

- **The object pools** are owned by `pool` and lent to everyone else by
  reference. `reactor` allocates sockets and timers from them; `deadlock`
  walks them; nobody else may free from them.
- **Stacks** are owned by `stack` and lent to `unit`. A stack returns to
  its pool only after `cancel` confirms the kernel holds no reference
  into it.
- **The actor context** is owned by the consumer (Limelight), installed
  by `sched` at mount time, and read by nothing else here.

## Hot paths

Named now so that measurement has a target later: the context switch
(`switch`), the park and wake pair (`unit`, `sched`), and completion
dispatch (`reactor`). No figures exist yet; `dev/BENCHMARKS.md` is
created with the first of them.
