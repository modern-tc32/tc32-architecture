# 06. Control Flow, Range Extension, Jump Tables, And Hazards

## Branch Forms

The control-flow instruction surface used for code generation is:

- short unconditional branch: `tj`
- short conditional branch: `tj<cc>`
- long call branch: `tjl`
- indirect transfer: `tjex reg`

## Range Rules

### Short Conditional Branch

- width: 16 bits
- range: `-256` to `+254` bytes from `pc + 4`

### Short Unconditional Branch

- width: 16 bits
- range: `-2048` to `+2046` bytes from `pc + 4`

### Long Call Branch

- width: 32 bits
- range: `-4194304` to `+4194302` bytes from `pc + 4`

## Safe Conditional Lowering

Rules:

- `safe to generate`: keep conditional control transfers in short form whenever possible
- `must not generate`: direct long conditional branches
- `toolchain obligation`: if a conditional target is out of range, rewrite the control flow so that the conditional edge remains short and the far transfer is moved onto an unconditional path

Safe pattern:

```asm
tjne .Lskip
tjl  far_target
.Lskip:
```

Alternative safe pattern:

```asm
tjeq .Lveneer
tj   .Lfallthrough
.Lveneer:
tjl  far_target
.Lfallthrough:
```

## Unsafe Condition Shapes

The following branch choices require explicit care:

- `tjpl` after arbitrary helper or callback return
- `tjls` in jump-table range-check lowering
- any long-form conditional branch

Preferred replacements:

- rewrite signed nonnegative tests using compare plus `tjlt`
- rewrite switch upper-bound checks using a condition other than `LS`

## Far Unconditional Control Flow

If a short `tj` cannot reach:

- `must not generate`: a direct long `tj` as the default compiler strategy
- `safe to generate`: a veneer or helper block that preserves the required architectural state

The implementation may choose any veneer design that preserves observable state and arrives at the final target correctly.

## Far Call Control Flow

If a direct `tjl` cannot reach:

- route the call through a long-call thunk or call veneer
- preserve call semantics, including return linkage

## Range-Extension Thunk For Calls

The canonical absolute long thunk is:

```asm
tpush {r0, r1}
tloadr r0, [pc, #16]
nop
nop
tstorer r0, [sp, #4]
nop
nop
tpop {r0, pc}
nop
nop
.word absolute_target
```

Properties:

- this thunk is entered by `tjl`
- the literal word holds the absolute code address
- fixed `nop` padding is part of the required sequence

## Jump-Table Policy

TC32 jump tables use a single canonical design.

Rules:

- `safe to generate`: inline 32-bit table entries of the form `.long target + 1`
- `must not generate`: byte-sized compressed jump tables
- `must not generate`: halfword-sized compressed jump tables

Canonical dispatch sequence:

```asm
tcmp   r0, #max_index
tjhi   .Ldefault
tshftl r0, r0, #2
tadd   r0, pc
tloadr r0, [r0, #8]
nop
nop
tmov   pc, r0
.align 2
.long case0 + 1
.long case1 + 1
.long case2 + 1
```

Notes:

- entries are 32-bit code pointers with bit 0 set
- the literal-load step uses the same `pc + 4` bias rule as other PC-relative instructions
- the extra `#8` compensates for the dispatch sequence layout

## Hazard Model

Two dependency hazards are mandatory in the TC32 code-generation contract:

- load-use hazard
- pop-use hazard

### Load-Use Hazard

If a register loaded by `tloadr` is consumed too soon:

- no intervening independent instruction: insert two `nop`
- one intervening independent instruction: insert one `nop`
- two or more intervening independent instructions: insert no `nop`

Example:

```asm
tloadr r0, [r1, #0]
nop
nop
tadd   r0, r2
```

### Pop-Use Hazard

If a register restored by `tpop` is consumed immediately:

- insert two `nop`

Example:

```asm
tpop {r0}
nop
nop
tadd r0, r1
```

## What The Toolchain Must Not Do

- do not pad after every memory instruction
- do not treat hazard padding as an assembler-only cleanup
- do not remove thunk `nop` padding
- do not change a safe short-branch rewrite back into a direct long conditional branch later in the pipeline
