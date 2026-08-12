# Context Switching

## What a switch is

A context switch moves execution between a worker's stack and a stackful
unit's stack. It saves whatever the calling convention says survives a
call, points the stack pointer at the other stack, and restores the same
set from there. Stackless units never switch: `poll` returns to its
caller, so the ordinary call sequence already saved what had to be saved
(`design/execution.md`).

Two things decide the cost. The first is the platform: an ABI names the
registers a call must preserve, and a switch preserves at least those.
The second is who compiled the frames below the suspension point, because
that decides whether the set can be narrowed.

## The full switch

The full switch saves everything the ABI calls callee-saved, and it is
correct without any knowledge of the code around it. It runs whenever the
narrow path does not apply.

| Platform | Callee-saved registers | Other state the switch carries |
|---|---|---|
| x86-64 SysV | `rbx`, `rbp`, `r12`–`r15` | the x87 control word and the MXCSR control bits are callee-saved; the status bits of both are not. No XMM register is callee-saved. |
| x86-64 Windows | `rbx`, `rbp`, `rdi`, `rsi`, `r12`–`r15` | `xmm6`–`xmm15` are non-volatile; the TEB's stack bounds must be swapped with the stack |
| AArch64 (ELF, Darwin) | `x19`–`x28`, `x29`, `x30` | the low 64 bits of `v8`–`v15`; Darwin reserves `x18` and it must not be touched |
| AArch64 Windows | as above | the low 64 bits of `v8`–`v15`, plus the TEB's stack bounds |
| ARM (32-bit) | `r4`–`r11`, `lr` | `d8`–`d15` |
| RISC-V 64 | `s0`–`s11`, `ra` | `fs0`–`fs11` |

### Windows carries more than registers

The thread environment block holds the stack bounds the operating system
and the compiler both consult: `StackBase`, `StackLimit` and
`DeallocationStack`. Switching to a unit's stack without swapping them
leaves the system believing the thread still runs on its original stack,
so guard-page growth, structured exception handling and any stack probe
consult the wrong range.

`DeallocationStack` is undocumented and its offset has moved between
Windows versions, so it is read from the running TEB rather than
hard-coded. Windows fibers exist because this bookkeeping is easy to get
wrong, and a switch that ignores it is not a lighter switch — it is a
broken one.

### Shadow stacks

Where CET user shadow stacks are enabled, each unit needs a shadow stack
of its own, and the switch swaps it with `RSTORSSP` and `SAVEPREVSSP`.
The hardware verifies a restore token at the top of the target shadow
stack, so the pair cannot be replaced by an ordinary move. Windows does
the same inside `SwitchToFiberContext`, reading the current shadow stack
pointer with `RDSSP` before the swap; QEMU's assembly coroutine backend
carries the same sequence.

This adds a second allocation per stackful unit, sized separately, and
`design/stacks.md` owns it.

## The narrow switch

When every frame from the suspension point down to the scheduler was
compiled by us, the switch may save less than the ABI demands. The
callee-saved registers hold values belonging to *our* frames, and our
compiler knows which of them are live across the suspension point. A mask
computed there lets the switch save the live ones and skip the rest.

The idea is Photon's context-aware calling stack: a switch at a known
caller site saves only what the site will need afterwards, which is what
brings a stackful switch close to a stackless one in cost.

### Where the mask comes from

The Limelight compiler emits one mask per suspension site, in a side
table keyed by the return address at that site. The switch reads the
mask, not the site: the same routine serves every site, and the table is
the same shape as the unwind table already needed for exceptions, which
is why it is keyed the same way.

A suspension site in code we did not compile has no entry, and its
absence is what selects the full switch.

## Choosing between them

One bit per unit selects the path: **a live foreign frame below the
current position**. It is set when execution crosses into code we did not
compile and cleared when that call returns.

The bit has a second reader. A unit with a live foreign frame is pinned
to its thread, because the foreign code holds thread-local state we
cannot move (`design/execution.md`). The same fact drives both: we do not
know what is below us, so we neither narrow the switch nor migrate the
unit.

Setting and clearing happen in one place, at the ABI boundary the
substrate already generates for foreign calls, so no rule has to be
remembered by hand.

### Why the boundary is the foreign frame and not the unit kind

A stackful unit whose whole chain is ours narrows; the same unit narrows
no longer once it calls into OpenSSL and suspends inside a callback. The
property is a state of the stack, not a property of the unit, which is
why it is a bit that changes rather than a field set at creation.

## Unwinding, debuggers and profilers

A switch that leaves no unwind metadata cuts every backtrace at the
suspension point, and a profiler sampling a unit's stack sees a root that
does not exist. The switch therefore emits the platform's standard
unwinding metadata, linking a unit's stack back to the frame that resumed
it. `corosensei` already does this and states that the resulting
backtraces work with debuggers and profilers.

The narrow switch inherits the same obligation. Saving fewer registers
does not license describing fewer: the metadata describes where a value
was put, and a value that was not saved is described as unchanged.

## What is taken and what is written

`corosensei` provides the full path — seven architectures across ELF,
Darwin, Windows and UEFI, with unwind metadata, panic propagation and a
public `Stack` trait for our own allocator (`dev/DECISIONS.md`,
2026-08-12). Its published figures are 1.8 ns to 18.8 ns per switch
depending on architecture and operation; those are its measurements, not
ours.

The narrow path is ours, because it needs a mask no general library can
have. It is written after the full path works and after measurement says
the difference is worth a second code path — not before.

## What must be measured before the narrow path is written

No claim about the narrow path enters this repository without these
numbers, taken on one machine, one build, several runs, reported as
medians (`dev/WORKFLOW.md`):

- the full switch, in nanoseconds, on our stack allocator rather than the
  library's default;
- the narrow switch on the same machine, with a mask that saves nothing,
  which is the ceiling of the possible gain;
- the mask distribution over real Limelight code: how many registers are
  actually live at a suspension site. A ceiling that only a rare site
  reaches does not justify the path.
- the cost of reading the side table, since a table lookup can cost more
  than the six stores it saves.

`dev/BENCHMARKS.md` does not exist yet and is created with the first of
these.

## Decided elsewhere

| Question | Document |
|---|---|
| where a unit's stack and shadow stack come from | `design/stacks.md` |
| what suspends, and what the switch is part of | `design/execution.md` |
| the unit slot the mask table is reached from | `design/pool.md` |

## Open questions

- **Does the narrow path survive inlining?** The mask is computed per
  suspension site, but a site inlined into several callers has several
  live sets. Either the table keys on the return address after inlining,
  which multiplies entries, or the mask is the union over callers, which
  narrows less. Undecided until the Limelight compiler emits the first
  one.
- **Shadow stack sizing.** A shadow stack holds one entry per call frame,
  so its size is a function of depth rather than of frame size, and the
  lazy-commit scheme for ordinary stacks does not obviously transfer.
  Owned by `design/stacks.md`, raised here.
- **Whether the foreign-frame bit can be maintained cheaply enough.** It
  is set and cleared around every foreign call, which is two stores on a
  path that otherwise has none. For a consumer whose calls are mostly
  foreign this is a cost with no return, since such a unit never narrows
  anyway. A per-unit opt-out is the obvious answer and is not yet
  designed.
