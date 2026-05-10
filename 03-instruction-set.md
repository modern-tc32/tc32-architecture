# 03. Instruction Set, Encodings, And Semantics

## Overview

TC32 instructions are encoded as either:

- one 16-bit halfword, or
- one 32-bit instruction composed of two consecutive halfwords

This chapter defines the canonical instruction families that a compiler, assembler, disassembler, and linker must agree on.

## Data Movement

Core data-movement instructions:

- register-immediate move: `tmov rd, #imm`
- register-register move: `tmov rd, rs`
- loads:
  - `tloadr`
  - `tloadrb`
  - `tloadrh`
  - `tloadrsb`
  - `tloadrsh`
- stores:
  - `tstorer`
  - `tstorerb`
  - `tstorerh`
- stack operations:
  - `tpush {...}`
  - `tpop {...}`
- indirect transfer:
  - `tjex reg`

Semantics:

- byte loads zero-extend or sign-extend according to the mnemonic
- halfword loads zero-extend or sign-extend according to the mnemonic
- word loads produce a full 32-bit value
- `tjex reg` transfers control to a code address held in a register
- loads, stores, `tpush`, `tpop`, and `tjex` leave the condition flags unchanged

## Arithmetic And Logical Operations

Core arithmetic and logical instructions:

- `tadd`
- `tsub`
- `taddc`
- `tsubc`
- `tcmp`
- `tcmpn`
- `tand`
- `tor`
- `txor`
- `tbclr`
- `tmul`

Generation rules:

- these operations are flag-producing in the canonical TC32 printing model
- `safe to generate`: low-register forms whenever available

## Flag Effects By Instruction Family

The table below is normative for compiler, assembler, disassembler, simulator, and hardware-facing validation work.

| Family | Flag effect | Notes for control flow |
| --- | --- | --- |
| `tmov rd, #imm8` | writes `N` and `Z`; preserves `C` and `V` | safe source for `EQ`, `NE`, and `MI`; not a safe source for unsigned-carry or signed-overflow branches |
| low-register `tmov rd, rs` | writes `N` and `Z`; preserves `C` and `V` | same control-flow rule as immediate `tmov` |
| high-register `tmov rd, rs` | flags unchanged | does not create a new branch condition |
| `tcmp a, b` and `tcmp a, #imm` | computes `a - b`; writes `N`, `Z`, `C`, `V`; writes no general register | primary safe source for signed and unsigned comparisons |
| `tcmpn a, b` | computes `a + b`; writes `N`, `Z`, `C`, `V`; writes no general register | use only when addition-based flag semantics are intentional |
| low-register `tadd` and `tsub` | write `N`, `Z`, `C`, `V` from the arithmetic result | safe source for range checks when the result-producing arithmetic is the intended compare |
| high-register `tadd` | flags unchanged | must not be used as an implicit compare |
| `taddc` | consumes old `C`; writes `N`, `Z`, `C`, `V` from `a + b + carry_in` | only generate when multiword arithmetic is intentional |
| `tsubc` | consumes old `C`; writes `N`, `Z`, `C`, `V` from `a - b - (1 - carry_in)` | only generate when multiword arithmetic is intentional |
| `tand`, `tor`, `txor`, `tbclr`, `tmovn` | write `N` and `Z` from the logical result; preserve `C` and `V` | safe source for `EQ`, `NE`, and `MI`; must not be treated as a fresh source for `HS`, `LO`, `HI`, `LS`, `GE`, `LT`, `GT`, or `LE` |
| `tmul` | writes `N` and `Z` from the low 32-bit result; preserves `C` and `V` | safe source only for `EQ`, `NE`, and `MI`; emit an explicit `tcmp` before any other condition use |
| `tshftl`, `tshftr`, `tasr`, `trotr` | write `N` and `Z` from the result; write `C` from the last shifted-out bit when the shift amount is nonzero; preserve `V` | safe source for `EQ`, `NE`, and `MI`; use carry-based conditions only when the shift carry-out is the intentional test |
| loads, stores, `tpush`, `tpop`, `tj`, `tjl`, `tjex`, `tmcsr`, `tmssr`, `tmrss`, `treti`, `nop` | flags unchanged | preserve any previously produced condition state |

Shift-specific carry rules:

- if a shift or rotate amount is zero, the instruction preserves the previous `C`
- if a shift or rotate amount is nonzero, `C` becomes the last bit shifted out
- `safe to generate`: use shifted-result flags mainly for `EQ`, `NE`, and `MI`
- `must not generate`: unsigned range branches from shift-produced `C` unless the exact bit-extract behavior is intentional

## Shift And Rotate Operations

- `tshftl`: logical left shift
- `tshftr`: logical right shift
- `tasr`: arithmetic right shift
- `trotr`: rotate right

Decode rule:

- `toolchain obligation`: decode the unsigned right-shift encoding before generic shift aliases so that `tshftr` is not printed as `tshftl`.

## High-Register Forms

Selected instructions support high registers such as `r8` and `lr`.

Examples of real high-register forms:

```asm
tmov r8, r1
tmov r1, r8
tcmp r8, r1
tadd r1, r8
```

These are real instruction forms, not pseudo-operations.

## Short Branches

### Short Unconditional Branch

- mnemonic: `tj`
- width: 16 bits
- relocation meaning: short branch
- displacement model: signed 11-bit halfword-scaled immediate
- address formula:

```text
target = branch_address + 4 + sign_extend(imm11 << 1)
```

Exact usable byte range:

- minimum displacement: `-2048`
- maximum displacement: `+2046`

Example:

```text
branch at 0x00000000
max forward short tj target = 0x00000802
```

### Short Conditional Branch

- mnemonic family: `tj<cc>`
- width: 16 bits
- displacement model: signed 8-bit halfword-scaled immediate
- address formula:

```text
target = branch_address + 4 + sign_extend(imm8 << 1)
```

Exact usable byte range:

- minimum displacement: `-256`
- maximum displacement: `+254`

Example:

```text
branch at 0x00000000
max forward short conditional target = 0x00000102
```

## Long Call Branch

- mnemonic: `tjl`
- width: 32 bits
- displacement model: signed 22-bit halfword-scaled immediate
- address formula:

```text
target = branch_address + 4 + sign_extend(imm22 << 1)
```

Exact usable byte range:

- minimum displacement: `-4194304`
- maximum displacement: `+4194302`

Example:

```text
branch at 0x00000000
max forward tjl target = 0x00400002
```

## Long Direct Branch Encoding

A 32-bit direct branch form corresponding to long `tj` exists in the machine-control and relocation model.

Classification:

- `architecturally defined`
- `must not generate` as the normal compiler lowering for far edges on TLSR8258-class hardware

If a far non-call edge cannot be expressed with short `tj`, route it through a veneer or other safe helper sequence.

## PC-Relative Literal Loads

Literal loads use the form:

```asm
tloadr rN, [pc, #imm]
```

Properties:

- width: 16 bits
- base: `pc + 4`
- immediate scale: 4-byte units
- effective address:

```text
literal_address = instruction_address + 4 + imm
```

Exact usable positive range:

- minimum offset: `0`
- maximum offset: `0x3f8`
- required alignment of the referenced word: 4 bytes

Example:

```text
instruction at 0x00000000
largest encodable literal word at 0x000003fc
encoded immediate = 0x3f8
```

## Stack Operations

### Push

- `tpush {reglist}` decrements `sp` and stores the listed registers.
- register ordering is the canonical ascending-register stack order.

### Pop

- `tpop {reglist}` loads the listed registers and increments `sp`.
- if `pc` is present, the instruction also returns.

Generation rules:

- `safe to generate`: ordinary callee-saved restore plus `pc`
- `must not generate`: multi-pop return forms that include `r3` and `pc` together

## Special Instructions

### `nop`

- semantic effect: no architecturally visible state change
- canonical 16-bit encoding example: `0x06c0`

### `treti`

- semantic effect: return from interrupt context
- canonical 16-bit encoding example: `0x6900`

### `tjex lr`

- semantic effect: indirect return through `lr`
- canonical 16-bit encoding example: `0x0770`

## Example Encodings

The following examples are useful for bring-up of encoders and decoders:

| Instruction | Example encoding |
| --- | --- |
| `nop` | `0x06c0` |
| `treti` | `0x6900` |
| `tjex lr` | `0x0770` |
| `tpush {r2, r3}` | `0x640c` |
| `tpop {r2, r3}` | `0x6c0c` |

These examples are not a substitute for a complete instruction decoder, but they anchor the canonical byte stream for early implementation testing.

## Instruction Families That Need Conservative Lowering

The following are machine-visible instruction families that require explicit compiler policy:

- short immediate add/sub operations on pointers
- long direct branch forms
- direct long conditional branches
- condition-sensitive branches using `PL`, `GE`, or `LS` when a safer rewrite exists

The rules for those cases are defined in later chapters and are mandatory for correct code generation.
