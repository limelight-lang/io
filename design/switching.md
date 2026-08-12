# Context Switching

## What a switch is

A context switch moves execution between a worker's stack and a stackful
unit's stack. It preserves whatever holds a live value for the side being
left, points the stack pointer at the other stack, and restores the same
from there. Stackless units never switch: `poll` returns to its caller,
so the ordinary call sequence already saved what had to be saved
(`design/execution.md`).

## The full switch

The full switch preserves everything the platform's calling convention
calls callee-saved, and it is correct without any knowledge of the code
around it. It is the default, and the alternative below narrows it only
under a condition that has to be built for.

| Platform | Preserved by the switch | Also carried |
|---|---|---|
| x86-64 SysV | `rbx`, `rbp`, `r12`–`r15` | the x87 control word and the MXCSR control bits, which the ABI makes callee-saved; the status bits of both are caller-saved. No XMM register is callee-saved. |
| x86-64 Windows | `rbx`, `rbp`, `rdi`, `rsi`, `r12`–`r15` | `xmm6`–`xmm15`, which are non-volatile here and not on SysV; the TEB fields below |
| AArch64 ELF | `x19`–`x28`, `x29`; `d8`–`d15` (low 64 bits of `v8`–`v15`) | `fpcr`; `x18` is the shadow call stack pointer on Android and must not be used as scratch |
| AArch64 Darwin | as above | `x18` is reserved by the platform and must not be touched |
| AArch64 Windows | as above | `x18` holds the TEB pointer; the TEB fields below |
| ARM 32-bit | `r4`–`r11`; `d8`–`d15` | `fpscr` |
| RISC-V 64 | `s0`–`s11`; `fs0`–`fs11` | `fcsr` |

The return address is not in this table. `x30`, `lr` and `ra` are
caller-saved in their conventions; a switch stores the resume address for
its own reasons, which is a different obligation and is described with
the saved context rather than with the preserved set.

### Windows carries the TEB

Four fields in the thread environment block describe the stack to the
operating system, the compiler's stack probes and structured exception
handling: `StackBase`, `StackLimit`, `DeallocationStack` and
`GuaranteedStackBytes`. All four are swapped with the stack. Swapping
three of them leaves a host that called `SetThreadStackGuarantee` with
wrong overflow behaviour on every unit stack.

Their offsets are not published, so they are hard-coded per architecture,
as `corosensei` does in its Windows stack code. Hard-coding an
undocumented offset is a liability rather than a technique: the offsets
are checked at startup against values read back from a known-good thread,
and a mismatch refuses to run rather than corrupting a TEB.

### Shadow stacks are ours to write

Where hardware shadow stacks are enforced, a unit needs one of its own
and the switch must move it. `corosensei` does not do this: its x86-64
code contains no CET sequence and its documentation claims none. The
claim that the library covers the whole full switch is therefore false on
any host with user shadow stacks enabled, and this is the first thing the
implementation owes.

What the switch needs, per platform:

- **x86-64 CET.** Swap the shadow stack pointer with `RSTORSSP` and
  `SAVEPREVSSP`. The hardware verifies a restore token at the top of the
  target shadow stack, so no ordinary move substitutes. A unit whose
  shadow stack is not switched dies at its first `ret` with a
  control-protection fault.
- **AArch64.** Android's shadow call stack lives in `x18` and is switched
  by swapping that register. ARMv9 guarded control stacks are a separate
  mechanism with its own instructions.

Allocation is the harder half and it does not belong to the switch. On
Linux a shadow stack comes only from `map_shadow_stack`, which places the
restore token at creation. Windows exposes no documented way to allocate
one for a context that is not a fiber. A pooled shadow stack cannot be
handed to a second unit until its token is reset, and with `WRSS`
normally disabled that means remapping rather than writing.
`design/stacks.md` owns this, and it owns a question rather than a
sizing table.

## Why a live-register mask does not narrow the switch

The obvious narrowing is wrong, and the reason is worth stating because
it is the reason the design took another road.

Let Limelight function A keep a value in `x19` and call B. B does not use
`x19`, so B leaves it alone, which satisfies B's ABI obligation without
spilling anything. B suspends. B's liveness at that site does not mention
`x19`, because the value is A's. A switch that saved only B's live
registers would leave `x19` unsaved; another unit runs on the worker and
overwrites it; A resumes and computes with a value belonging to a
different unit. Nothing crashes and nothing is diagnosable.

The sound per-site set is "the registers live in B, plus every
callee-saved register B did not spill" — and a function that spills
little leaves nearly the full set, which is what the narrowing was
supposed to avoid. Per-site liveness cannot see the chain above it, so it
cannot be the answer.

## The narrow switch is a calling convention, not a mask

The switch preserves fewer registers only when there is nothing in them
to preserve. That is a property of how the suspendable path was compiled,
and it is arranged rather than detected.

**Along a suspendable path, Limelight compiles with a convention that has
no callee-saved registers** — the shape of clang's `preserve_none`. Every
function on such a path spills its own live values before a call that may
suspend, so at the suspension point no register holds anything for any
frame in the chain. The switch then moves the stack pointer and the
resume address and nothing else.

The cost does not vanish; it moves. The compiler spills what is actually
live at that point instead of the switch spilling a fixed set, and that
is the whole gain: a site with two live values costs two stores rather
than seven. Whether the average site is cheap enough to pay for a second
code path is a measurement, listed below, and it has not been taken.

### The park primitive must be on that path

Parking runs through substrate code — the state store, the record write,
the arming of each half (`design/execution.md`) — and that code is
compiled by rustc under the ordinary convention. Its frames hold
callee-saved values like any others, so a switch reached through them
cannot narrow.

The path from the suspension point to the switch instruction is therefore
written to hold nothing: the tail of the park primitive is assembly, or a
naked function, and the parking work happens before it. Any design that
leaves an ordinary compiled frame between the site and the switch pays
the full save, and the narrow path is not available to it.

### Half the switch is never narrow

The worker's side of every switch is rustc-compiled and always preserves
the full set. The narrowing applies to the unit's half alone, so it can
remove at most half of the switch's register traffic. Any expectation set
against a stackless resume has to start from that ceiling.

### The narrow path still swaps everything that is not a register

The TEB fields and the shadow stack are swapped on both paths, without
exception. They describe the stack, not the frames, and a narrow switch
that skipped them would produce exactly the corruption the full-switch
section describes.

## Selecting the path: the foreign-frame counter

Selection is per unit and dynamic, because the property is a state of the
stack rather than a property of the unit. A unit whose whole chain is
ours narrows; the same unit stops narrowing once it calls into a foreign
library and suspends in a callback.

**The marker is a depth counter, not a bit.** The stub around a foreign
call increments on entry and decrements on return. A single bit fails on
re-entry: our code calls OpenSSL, OpenSSL calls our callback, the
callback makes any foreign call of its own, and that call's return clears
the bit while OpenSSL's frames are still live below. Everything that
reads the marker then reads it wrong.

Its readers are three, and all of them are safety-critical:

| Reader | What a wrong value costs |
|---|---|
| the switch | narrows with foreign frames live: corrupted foreign registers |
| migration (`design/execution.md`) | steals a unit holding `errno` and other thread-local state |
| forced teardown (`design/execution.md`) | unwinds through a frame not compiled for it |

Because the last two readers are about safety rather than speed, a unit
that opts out of maintaining the counter does not opt out of its value:
its counter reads as permanently non-zero, so it never narrows, never
migrates and is never force-killed. Opting out buys back two instructions
per foreign call and costs work stealing, which is the right trade only
for a consumer whose calls are nearly all foreign.

## The saved context, the restore, and the first resume

**Where the context lives.** At the top of the unit's own stack, not in
the unit slot. Its size is fixed per platform, because the narrow path
narrows by having nothing to save rather than by saving a variable
subset — which is a second reason to prefer the convention over the mask,
since a variable-size context would make the slot layout in
`design/pool.md` depend on a compiler's output.

**The restore.** Symmetric with the save and driven by the same platform
constant. It does not have to learn what the save chose, because both
sides of one switch are the same code path.

**The first resume.** A unit that has never run has no saved context to
restore, so its stack is laid out at creation: a synthetic frame whose
return address is the unit's entry trampoline, a shadow-stack restore
token where hardware shadow stacks are enabled, and an unwind root that
terminates the backtrace rather than pointing into a caller that does not
exist.

## Unwinding, debuggers and profilers

A switch that leaves no unwind metadata cuts every backtrace at the
suspension point, and a profiler sampling a unit's stack sees a root that
does not exist. The switch emits the platform's standard unwinding
metadata, linking a unit's stack back to the frame that resumed it.
`corosensei` does this and states that the resulting backtraces work with
debuggers and profilers.

The narrow path inherits the obligation and satisfies it trivially: under
a convention with no callee-saved registers there is no saved location to
describe, and the values a debugger wants were spilled by the compiled
code and are described by its own metadata.

## What is taken and what is written

`corosensei` provides the register and stack-pointer half of the full
switch across seven architectures on ELF, Darwin, Windows and UEFI, with
unwind metadata, panic propagation and a public `Stack` trait
(`dev/DECISIONS.md`, 2026-08-12). Its README publishes 1.8 ns to 18.8 ns
per switch depending on architecture and operation; those are its
figures, taken on its machines, and this repository has none of its own.

Written here: the CET and shadow-call-stack half, which the library does
not have; the foreign-frame counter and its stub; the narrow path,
including the assembly tail of the park primitive; and the first-resume
layout, to the extent the library's own entry sequence does not cover it.

## What must be measured before the narrow path is written

No claim about the narrow path enters this repository without these
numbers, taken on one machine, one build, several runs, reported as
medians (`dev/WORKFLOW.md`):

- the full switch on our stack allocator rather than the library's
  default;
- the same switch with the unit half preserving nothing, which is the
  ceiling of the possible gain and is bounded by the worker half above;
- the spill count the `preserve_none` convention adds to real Limelight
  code at and around suspension sites, since the gain is a transfer and
  the transfer can lose;
- the cost of the foreign-call stub, two memory operations on a path that
  otherwise has none.

`dev/BENCHMARKS.md` does not exist yet and is created with the first of
these.

## Decided elsewhere

| Question | Document |
|---|---|
| stack and shadow-stack allocation, pooling and release | `design/stacks.md` |
| what suspends, parks and migrates | `design/execution.md` |
| the unit slot, which does not hold the saved context | `design/pool.md` |

## Open questions

- **Whether `preserve_none` composes with the rest of Limelight's
  codegen.** The convention is available in clang and reachable from LLVM
  IR, but a path compiled with it cannot be called from an ordinary path
  without a thunk, and every suspendable function is on it. The cost of
  the thunks at the boundary is not estimated.
- **arm64e pointer authentication.** A signed `x30` is authenticated
  against the stack pointer it was signed with, so restoring it under a
  different context faults. Whether `corosensei` handles arm64e, and what
  the switch owes there, is not established.
- **Windows shadow-stack allocation for a non-fiber context.** Stated
  above: no documented API. If none exists, either units on Windows run
  as OS fibers where CET is enforced, or CET hosts get the full switch
  with a per-thread shadow stack and no unit-local one — both change the
  design, and neither is chosen.
- **ARMv9 guarded control stacks.** A third mechanism after CET and
  Android's shadow call stack, with its own instructions and its own
  allocation story. Not investigated.
