# Stacks

## What a stack costs

A stackful unit's stack is a reservation of address space plus the pages
it actually touches. The reservation is large and cheap; the touched
pages are what the process pays for. Stackless units have no stack of
their own (`design/execution.md`).

Measured on this project's Linux machine — 6.6 under WSL2, 4 KB pages,
`ulimit -s` 8192 KB, overcommit at the kernel default — with an ordinary
anonymous read-write mapping per stack and a guard page below it:

| Reservation | Pages touched per stack | RSS per stack | Address space |
|---|---|---|---|
| 2 MB | 1 | 4.0 KB | 2 MB |
| 2 MB | 16 | 64.0 KB | 2 MB |

Ten thousand such stacks cost 40 272 KB of resident memory against 20 GB
of address space. The reservation is not the number to design around; the
touched depth is.

**These are 4 KB-page figures and they do not transfer.** Apple Silicon
runs 16 KB pages and Android is moving to them, so the same measurement
there gives 16.0 KB per one-page stack, and ten thousand of them cost
160 MB rather than 40 MB. Everything below is stated in pages for this
reason, and every byte figure names the page size it was taken at.

## Lazy commit needs no machinery, on two of three platforms

An anonymous mapping is zero-fill-on-demand on Linux and on Darwin, so
reserving a stack with `mmap(PROT_READ | PROT_WRITE, MAP_NORESERVE)`
commits pages only as they are touched. The table above is that mapping,
unassisted.

**The Linux figure assumes overcommit is not disabled.** `MAP_NORESERVE`
is honoured only while `vm.overcommit_memory` is 0 or 1; under 2 the
kernel charges every reservation whole, so 10 000 stacks of 2 MB charge
20 GB of commit and allocation fails in the hundreds. A host in that mode
runs with a class table sized in tens of kilobytes rather than megabytes,
and the substrate reads the sysctl at startup rather than discovering the
failure under load.

**Windows is not this shape.** A stack there is reserved with
`VirtualAlloc(MEM_RESERVE)` and grown through a `PAGE_GUARD` page, and
whether the kernel's automatic growth applies to a reservation we made
ourselves once its bounds are installed in the TEB
(`design/switching.md`) is **not verified**. Two fallbacks exist if it
does not: commit the whole class up front, which ends lazy commit on
Windows, or commit the faulting page from a vectored exception handler.
The second is the same mechanism as the Linux experiment below, and its
cost has never been measured on Windows, so no cost model for Windows
stacks exists yet.

The corrected form of segmented growth was measured once and is recorded
because knowing it collapses into plain reservation is what closes the
question (`dev/DECISIONS.md`, 2026-08-12): a stack reserved `PROT_NONE`
and grown a page at a time from a `SIGSEGV` handler ran 1501 frames with
200 faults and 800 KB touched of 2 MB, with no prologue check anywhere.
It works and it buys nothing over the ordinary mapping on Linux.

## Nothing the kernel touches lives on a stack

**An I/O buffer, an `iovec` array, a `msghdr`, and every other structure
the kernel may read or write after submission come from the buffer pool,
never from a unit's stack.** This is a rule of the substrate, enforced by
the only API that submits operations (`design/reactor.md`).

The rule exists because completion-based I/O separates "the call
returned" from "the kernel is done". A read whose buffer is a local
variable stays live until its completion arrives, and cancelling the unit
does not cancel that. Without the rule, three failures follow and none of
them is detectable:

- the unit dies, its stack is pooled, and the late completion writes into
  the next unit's frames;
- the unit does not die but returns past the awaiting frame, and the late
  completion writes into its own live frames — the same corruption
  without any reuse;
- the retired loser of an `await` with a timeout completes after the
  winner resumed, which is the common case rather than the exotic one
  (`design/execution.md`).

With the rule, a stack's lifetime is exactly its unit's, and this
document needs no protocol for a stack that outlives its owner. That is
the whole reason the rule is stated here rather than only in the reactor:
it is what makes stack management simple, and giving it up would put a
per-stack in-flight count, an owner for orphaned stacks, and a policy for
completions that never arrive back into this design.

## The ceiling is mappings, not memory

Each stack with its own guard page occupies two kernel mappings, and the
kernel bounds their number. Measured on the same machine, where
`vm.max_map_count` is 65530: allocation failed after **32 754 stacks**,
with the process holding 65 531 mappings. Without per-stack guard pages
the same loop ran past that point and was killed by the out-of-memory
killer instead.

Two ways past it, and they are not equivalent:

- **Raise `vm.max_map_count`.** One line of deployment configuration,
  unavailable where we do not control the host.
- **Carve stacks from slabs.** One mapping holds many stacks, so the
  count stops scaling with the unit count. What is given up is the guard
  page, and with it the ability to catch an overflow at the instruction
  that caused it.

The default is one mapping and one guard page per stack. A consumer above
thirty thousand simultaneous stackful units chooses slabs explicitly, and
**slabs are refused for any unit that enters foreign code**: a large
foreign frame that steps over a red zone corrupts the *neighbouring*
unit's stack while leaving its own red zone intact, so the check on
return to the scheduler passes and reports nothing.

## Guard bands must match the probe interval

A frame larger than the guard band steps past it and writes into whatever
lies below — the stack clash. A one-page band is defeated by a function
with a 64 KB local array in a single `sub sp`.

The defence is probing, and probing only works when the compiler's
assumed guard size matches ours. GCC's documented default for
`--param stack-clash-protection-guard-size` is 64 KB on AArch64, so
gcc-compiled AArch64 code probes at 64 KB intervals and a 32 KB frame
steps clean over a 4 KB band. The band is therefore **per platform, sized
to the largest probe interval of the compilers that produce code running
on our stacks**, not one page everywhere.

Three facts about probing that the design has to carry rather than
assume:

- `gcc -fstack-clash-protection` exists on this project's x86-64 machine,
  verified by compiling with it. That says nothing about AArch64 defaults
  or about other compilers, and the interval above is from GCC's
  documentation rather than from a measurement here.
- rustc's probe support is per target, and whether it covers 32-bit ARM
  is **not verified**. The substrate's own code is what would go
  unprobed.
- The frames of libraries below our units — OpenSSL, libxml, a PHP
  extension — are probed by nobody's declaration. Widening the band is
  the only defence that reaches them, which is why the band is sized by
  platform rather than declared per consumer.

## Size classes

Stacks come in a small fixed set of sizes, because touched depth varies
by orders of magnitude between a unit awaiting a socket and a unit
running a recursive parser. A class is chosen at creation from a hint the
consumer supplies, and the unit keeps that stack for its life: there is
no growth, by decision (`dev/DECISIONS.md`, 2026-08-12).

The hint crosses the C ABI as a size in bytes, rounded up to the nearest
class; out of range clamps to the largest and is reported once per
process; absent takes the default class.

Class sizes are a deployment parameter and a page-size-dependent one, so
there is one table per page size and none of them is fixed here. What is
fixed is the shape: a small number of classes, each a power-of-two
multiple of the page size, each with its own free list.

## Pooling and release

A stack returns to the free list of its class when its unit completes.
Because nothing the kernel touches lives on a stack, completion is the
only condition: there is no confirmation to wait for.

**Release is ordered, and one order is corrupting.** The pages are
dropped first and the stack is published to the free list second.
Publishing first lets another worker pop the stack and write frames into
it before the drop discards those very pages, and the second worker then
reads zeros out of memory it just wrote.

**Release is not per completion.** Dropping pages costs a syscall and a
TLB shootdown, and `design/execution.md` fixes one unit per message, so
releasing on every completion would put both on the per-message path.
Stacks return to the free list with their pages intact, and pages are
dropped only from stacks above a per-class warm watermark, in batches.
The watermark is a number nobody has measured; the design fixes that it
exists.

Per platform, dropping pages means:

| Platform | Call | What it releases |
|---|---|---|
| Linux | `madvise(MADV_DONTNEED)` | resident pages; the mapping and the reservation stay |
| Darwin | `madvise(MADV_FREE_REUSABLE)`, with `MADV_FREE_REUSE` before the range is touched again | resident pages and `phys_footprint`, which is what iOS jetsam counts |
| Windows | `VirtualAlloc(MEM_DECOMMIT)` | the commit charge; re-taking the stack must recommit |

**Windows differs and the difference is a leak if it is missed.**
`MEM_RESET` does not release anything: it licenses the system to discard
contents rather than page them out, and the pages stay committed and
charged. Ten thousand pooled stacks that each once touched 64 KB would
hold 640 MB of commit charge while idle. `MEM_DECOMMIT` is the release,
and it costs a recommit on re-take, which is why the warm watermark
matters more on Windows than elsewhere.

## Overflow

Overflow faults on the guard page. **The default outcome is that the
process ends**, and this document states that rather than the earlier
promise of unwinding the unit, because unwinding from a fault is not
generally available:

- the handler needs `sigaltstack`, since the stack pointer is in the
  guard page;
- unwinding must be async-signal-safe, and Rust's panic machinery
  allocates and takes locks, so a fault taken inside the allocator
  deadlocks in the handler;
- unwinding from an arbitrary instruction needs asynchronous unwind
  tables, and ARM EHABI defines unwinding only at call sites, so on
  32-bit Android it cannot be represented at all;
- landing pads run destructors near the guard, and a second fault during
  unwind ends the process anyway — this is what
  `SetThreadStackGuarantee` reserves headroom for on Windows, and nothing
  analogous is reserved here;
- where CET is enabled the shadow stack must be unwound with `INCSSP`
  alongside.

**Where it can be done, it is an opt-in per platform, not a promise.** On
x86-64 Linux the list above is achievable: alternate signal stack,
asynchronous tables, reserved headroom, and a handler that distinguishes
a guard fault from any other `SIGSEGV` by address. On Windows the
mechanism is different in kind — `STATUS_STACK_OVERFLOW` through
structured exception handling, with `_resetstkoflw` to re-arm the guard —
and is a separate implementation rather than a port.

A unit with a live foreign frame is never unwound, on any platform
(`design/execution.md`).

The consumer is told which unit died and how deep it was through the
death-notification path of `io-api`, which no document owns yet: a unit
that dies unwaited-for has no waker to carry a result.

## Shadow stacks

Where hardware shadow stacks are enforced, each stackful unit needs one,
allocated with it and switched with it (`design/switching.md`). Almost
nothing above transfers to them, and the section exists to say so.

- **Allocation is not `mmap`.** On Linux a user shadow stack comes only
  from `map_shadow_stack`, which plants the restore token at creation;
  ordinary anonymous memory cannot be pivoted to with `RSTORSSP`. Windows
  documents no way to allocate one for a context that is not a fiber.
- **Pooling does not work as written.** Dropping a shadow stack's pages
  destroys the restore token the next `RSTORSSP` verifies, and replanting
  it needs `WRSS`, which is itself a security opt-in. A pooled shadow
  stack is therefore remapped rather than reset, which is a syscall per
  reuse.
- **Sizing is by depth**, since a shadow stack holds one return address
  per frame, so it needs a class table of its own that does not follow
  the ordinary one.
- **The ceiling halves.** A unit costs two reservations, and with guard
  pages four mappings, so the 32 754 measured above becomes roughly half
  that where CET is on. Whether a guard page can be `mprotect`ed into a
  shadow-stack mapping at all is **not verified**.

This is a set of unknowns rather than a design, and it is the first thing
the Windows and CET backends must resolve.

## The worker stack

Worker threads run on ordinary thread stacks, and stackless units consume
them (`design/execution.md`). Their size is the platform default unless
the consumer overrides it at startup, and overflow there is not a unit's
overflow: the frames on it belong to the scheduler, so the process ends
with no attempt at recovery. A consumer whose stackless units nest deeply
raises the worker stack size rather than relying on a per-unit mechanism
that does not apply.

## What is taken

`corosensei` exposes a public `Stack` trait, so this allocator plugs in
under the switch without forking the library (`dev/DECISIONS.md`,
2026-08-12). Its `DefaultStack` is not used: it allocates per stack, and
pooling is the reason this document exists.

## Decided elsewhere

| Question | Document |
|---|---|
| what a switch saves, the TEB swap, the shadow-stack swap | `design/switching.md` |
| which units have stacks, and why a unit always ends itself | `design/execution.md` |
| the buffer pool that holds everything the kernel touches | `design/reactor.md` |
| the coroutine object that points at a stack | `design/execution.md` |

## Open questions

- **The class tables.** Sizes, their number, and the warm watermark are
  measurements, one table per page size, and no measurement exists.
- **Windows lazy commit.** Whether TEB-installed bounds give a
  self-allocated reservation the kernel's guard growth, and what the
  fallback costs.
- **Shadow stacks entirely.** Allocation on Windows, pooling on Linux,
  guard pages, and the depth-driven class table.
- **32-bit Android.** A 2 MB class against a two- to three-gigabyte user
  address space caps a process near a thousand stackful units, and the
  only lever is smaller classes: actors are stackful by decision and
  C-ABI consumers get stackful units only, so there is no escape into the
  stackless kind for the consumers that exist there.
- **Probe coverage.** Whether rustc probes on every target we ship, and
  what the substrate does on a target where it does not.
