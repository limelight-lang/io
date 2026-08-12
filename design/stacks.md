# Stacks

## What a stack costs

A stackful unit's stack is a reservation of address space plus the pages
it actually touches. The reservation is large and cheap; the touched
pages are what the process pays for. Stackless units have no stack of
their own and appear here only where they consume the worker's
(`design/execution.md`).

Measured on this project's Linux machine (6.6, WSL2, 4 KB pages,
`ulimit -s` 8192 KB), with an ordinary anonymous read-write mapping per
stack and a guard page below it:

| Reservation | Pages touched per stack | RSS per stack | Address space |
|---|---|---|---|
| 2 MB | 1 | 4.0 KB | 2 MB |
| 2 MB | 16 | 64.0 KB | 2 MB |

Ten thousand such stacks cost 40 272 KB of resident memory against 20 GB
of address space. The reservation is therefore not the number to design
around; the touched depth is.

## Lazy commit needs no machinery

An anonymous mapping is zero-fill-on-demand on Linux and on Darwin, so
reserving a stack with `mmap(PROT_READ | PROT_WRITE, MAP_NORESERVE)`
already commits pages only as they are touched. The table above is that
mapping, unassisted.

A separate experiment on the same machine confirmed the stronger form as
well: a stack reserved `PROT_NONE`, grown a page at a time from a
`SIGSEGV` handler that maps the faulting page and returns, ran 1501
frames with 200 faults and 800 KB touched of 2 MB reserved, with no
prologue check anywhere. It works, and it buys nothing over the ordinary
mapping, so the design does not use it. It is recorded because it is the
corrected form of segmented growth, and knowing that it collapses into
plain reservation is what closes that question
(`dev/DECISIONS.md`, 2026-08-12).

## The ceiling is mappings, not memory

Each stack with its own guard page occupies two kernel mappings, and the
kernel bounds their number. Measured on the same machine, where
`vm.max_map_count` is 65530: allocation failed after **32 754 stacks**,
with the process holding 65 531 mappings. Without per-stack guard pages
the same loop ran past that point and was killed by the out-of-memory
killer instead, which is to say mappings stopped being the limit.

Two ways past it, and they are not equivalent:

- **Raise `vm.max_map_count`.** One line of deployment configuration, no
  design cost, and unavailable where we do not control the host — a
  container image can carry the sysctl only if the platform allows it.
- **Carve stacks from slabs.** One mapping holds many stacks, so the
  mapping count stops scaling with the unit count. The guard page is what
  is given up: overflow is then detected by a check when the unit returns
  to the scheduler, which finds the damage after it happened rather than
  at the instruction that caused it.

The default is one mapping and one guard page per stack, because the
guard page catches overflow at the faulting instruction and that is worth
more than the ceiling for every workload below thirty thousand
simultaneous stackful units. A consumer above it chooses slabs
explicitly.

## Guard pages do not catch every overflow

A frame larger than the guard band can step past it and write into
whatever lies below — the stack clash. A guard page is one page; a
function with a 64 KB local array skips it in a single `sub rsp`.

Two defences, and the design uses both:

- **Probe large frames.** Code we compile probes every page of a frame
  larger than the guard band, which is what `gcc -fstack-clash-protection`
  does (available on this machine's gcc; verified by compiling with it).
  This is the prologue check that segmented stacks needed everywhere,
  reduced to frames above a page — a rare shape in generated code.
- **Widen the band where we cannot compile.** A consumer whose frames we
  do not control gets a guard band sized to the largest frame it declares,
  and the default band for undeclared code is one page, which is the
  status quo of every C program.

## Size classes

Stacks come in a small fixed set of sizes rather than one size, because
the touched depth varies by three orders of magnitude between a unit that
awaits a socket read and a unit that runs a recursive parser. A class is
chosen at creation from a hint the consumer supplies, and the unit keeps
that stack for its life: there is no growth, by decision
(`dev/DECISIONS.md`, 2026-08-12).

Classes are a deployment parameter, not a constant of the design. What
fixes them is measurement of real workloads, which does not exist yet.
What the design fixes is the shape: a small number of classes, each a
power-of-two multiple of the page size, each with its own free list.

## Pooling and release

A stack returns to the free list of its class when its unit completes.
Returning it does two things:

- **Drop the resident pages** with `madvise(MADV_DONTNEED)` on Linux,
  `madvise(MADV_FREE_REUSABLE)` on Darwin, or `VirtualAlloc(MEM_RESET)`
  on Windows. The mapping survives, so the mapping count does not
  fluctuate; the pages return to the system.
- **Keep the reservation**, so the next unit of that class takes a stack
  without a syscall.

**A stack is released only after every operation that named it has
confirmed that no kernel reference into it remains.** A submitted
io_uring read whose buffer lives on the unit's stack owns that memory
until its completion arrives, and cancelling the unit does not cancel
that ownership. `design/cancellation.md` owns the protocol; this document
owns the consequence, which is that a stack can outlive its unit.

## Overflow

Overflow is a fault, caught at the guard page, and it ends the unit
rather than the process. The unit is unwound from the faulting frame the
way a panic unwinds, its stack is released, and the consumer is told
which unit died and how deep it was.

Where the unit has a live foreign frame, unwinding is not available and
the process ends instead (`design/execution.md`). That is a real gap
rather than a policy choice: unwinding through a frame that was not
compiled to allow it is undefined, and the alternative to ending the
process is continuing with a corrupted stack.

## Shadow stacks

Where CET user shadow stacks are enabled, each stackful unit needs a
shadow stack of its own, allocated with it and switched with it
(`design/switching.md`). Its size is a function of call *depth* rather
than frame size, since it holds one return address per frame, so it does
not scale with the ordinary stack's class and needs a class table of its
own.

The lazy-commit argument transfers, because a shadow stack is also
touched only as deep as the unit goes. The mapping-count argument
transfers too, and doubles: a unit now costs two reservations and, with
guard pages, four mappings. Where CET is on, the ceiling measured above
halves.

## Per-platform mechanism

| Platform | Reserve | Commit | Release |
|---|---|---|---|
| Linux | `mmap` anonymous, `MAP_NORESERVE`, guard page via `mprotect` | on touch | `madvise(MADV_DONTNEED)` |
| Darwin (macOS, iOS) | `mmap` anonymous, guard page via `mprotect` | on touch | `madvise(MADV_FREE_REUSABLE)` |
| Windows | `VirtualAlloc(MEM_RESERVE)` plus a `PAGE_GUARD` page | pages committed as the guard page moves down | `VirtualAlloc(MEM_RESET)` |

Windows differs in kind: the automatic growth of a stack is driven by the
guard page and by the stack bounds recorded in the TEB, which a context
switch swaps (`design/switching.md`). Whether that growth applies to a
mapping we allocated ourselves once its bounds are installed in the TEB
is **not verified here** and must be confirmed against a running system
before the Windows backend is written.

### 32-bit targets

On 32-bit Android the reservation stops being free: a 2 MB class against
a user address space of two to three gigabytes caps the process at
roughly a thousand stackful units, well below the mapping ceiling that
bounds 64-bit hosts. Such a target takes a smaller class table, and the
consumer that needs many units there uses stackless units instead, which
is where the two-kind design pays.

## What is taken

`corosensei` exposes a public `Stack` trait, so this allocator plugs in
under the switch without forking the library (`dev/DECISIONS.md`,
2026-08-12). Its own `DefaultStack` is not used: it allocates per stack,
and the pooling and release protocol above is the reason this document
exists.

## Decided elsewhere

| Question | Document |
|---|---|
| what a switch saves, and the TEB and shadow-stack swap | `design/switching.md` |
| which units have stacks and how they suspend | `design/execution.md` |
| when the kernel stops holding a reference into a stack | `design/cancellation.md` |
| where the stack pointer lives in a unit slot | `design/pool.md` |

## Open questions

- **The class table.** Sizes and their number are a measurement, and the
  measurement needs real workloads. Until then the implementation carries
  one class and a comment.
- **Windows automatic growth on a mapping we own.** Stated above as
  unverified.
- **Slab layout.** The slab alternative removes per-stack guard pages, so
  overflow detection moves to a check on return to the scheduler. What
  that check costs, and whether a red zone at each stack's low end
  recovers most of the guard page's precision, is not designed.
- **Shadow stack class table.** Depth-driven rather than size-driven, so
  it cannot reuse the ordinary table, and no measurement exists.
