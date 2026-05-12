# 12. Complete Instruction Encoding Reference

This chapter is the normative bit-level reference for the TC32 instruction surface defined by this specification.

## Register Encoding

General-purpose register numbers:

| Register | Encoding |
| --- | --- |
| `r0` | `0` |
| `r1` | `1` |
| `r2` | `2` |
| `r3` | `3` |
| `r4` | `4` |
| `r5` | `5` |
| `r6` | `6` |
| `r7` | `7` |
| `r8` | `8` |
| `r9` | `9` |
| `r10` | `10` |
| `r11` | `11` |
| `r12` | `12` |
| `sp` | `13` |
| `lr` | `14` |
| `pc` | `15` |

Low-register forms accept only encodings `0` through `7`.

## Encoding Conventions

- All instruction words are stored little-endian in memory.
- For 16-bit instructions, bit positions are numbered within the 16-bit word.
- For 32-bit instructions, the low halfword occupies the lower-addressed 16 bits.
- Branch immediates are encoded as halfword-scaled displacements relative to `pc + 4`.
- Code pointers stored in data use `address + 1`.

## Flag Side-Effect Reference

This section ties encoding families to condition-state behavior.

| Encoding family | Flag effect |
| --- | --- |
| `tmov Rd, #imm8` | writes `N` and `Z`; preserves `C` and `V` |
| low-register `tmov Rd, Rs` | writes `N` and `Z`; preserves `C` and `V` |
| high-register `tmov Rd, Rs` | flags unchanged |
| high-register `tadd Rd, Rs` | flags unchanged |
| `tcmp`, `tcmp #imm`, high-register `tcmp` | write `N`, `Z`, `C`, `V` from subtraction |
| `tcmpn` | writes `N`, `Z`, `C`, `V` from addition |
| low-register `tadd` and `tsub` | write `N`, `Z`, `C`, `V` |
| `taddc` and `tsubc` | read old `C`; write `N`, `Z`, `C`, `V` |
| `tand`, `tor`, `txor`, `tbclr`, `tmovn` | write `N` and `Z`; preserve `C` and `V` |
| `tmul` | writes `N` and `Z`; preserves `C` and `V` |
| `tshftl`, `tshftr`, `tasr`, `trotr` | write `N` and `Z`; write `C` from shift/rotate carry-out; preserve `V` |
| loads, stores, `tpush`, `tpop`, `tj`, `tj<cc>`, long direct branches, `tjl`, `tjex`, `tmcsr`, `tmssr`, `tmrss`, `treti`, `nop` | flags unchanged |

Control-flow rule:

- `safe to generate`: branch conditions that require `C` or `V` only when the immediately preceding flag source is an explicit compare or arithmetic operation chosen for that purpose
- `must not generate`: unsafe `PL`, `LS`, and `GE` primary lowerings when an equivalent safer CFG rewrite exists

## Fixed 16-Bit Encodings

| Instruction | Encoding |
| --- | --- |
| `nop` | `0x06c0` |
| `treti` | `0x6900` |

## Special Register Instructions

### `tmcsr Rd`

```text
encoding = 0x6BC0 | Rd
constraints: Rd is low register
```

### `tmssr Rd`

```text
encoding = 0x6BD0 | Rd
constraints: Rd is low register
```

### `tmrss Rd`

```text
encoding = 0x6BD8 | Rd
constraints: Rd is low register
```

## Register Move Forms

### Low-to-Low `tmov Rd, Rs`

```text
encoding = 0xEC00 | (Rs << 3) | Rd
constraints: Rd < 8, Rs < 8
```

### General `tmov Rd, Rs`

If either register is high:

```text
encoding = 0x0600
         | ((Rd & 0x8) ? 0x80 : 0)
         | ((Rs & 0x8) ? 0x40 : 0)
         | ((Rs & 0x7) << 3)
         |  (Rd & 0x7)
constraints: Rd < 16, Rs < 16
```

## High-Register Add And Compare

### `tadd Rd, Rs` with high-register form

```text
encoding = 0x0400
         | ((Rd & 0x8) ? 0x80 : 0)
         | ((Rs & 0xF) << 3)
         |  (Rd & 0x7)
constraints: Rd < 16, Rs < 16
```

### `tcmp Rd, Rs` with high-register form

```text
encoding = 0x0500
         | ((Rd & 0x8) ? 0x80 : 0)
         | ((Rs & 0xF) << 3)
         |  (Rd & 0x7)
constraints: Rd < 16, Rs < 16
```

## Immediate Move And Compare

### `tmov Rd, #imm8`

```text
encoding = ((0xA0 + Rd) << 8) | imm8
constraints: Rd < 8, 0 <= imm8 <= 255
```

### `tcmp Rs, #imm8`

```text
encoding = ((0xA8 + Rs) << 8) | imm8
constraints: Rs < 8, 0 <= imm8 <= 255
```

## Register Compare Forms

### `tcmp Ra, Rb`

```text
encoding = 0x0280 | (Rb << 3) | Ra
constraints: Ra < 8, Rb < 8
```

### `tcmpn Ra, Rb`

```text
encoding = 0x02C0 | (Rb << 3) | Ra
constraints: Ra < 8, Rb < 8
```

## Three-Operand Arithmetic

### `tadd Rd, Rs, Rb`

```text
Enc = Rd | (Rs << 3) | (Rb << 6)
encoding = ((0xE8 + (Enc >> 8)) << 8) | (Enc & 0xFF)
constraints: Rd < 8, Rs < 8, Rb < 8
```

### `tsub Rd, Rs, Rb`

```text
Enc = Rd | (Rs << 3) | (Rb << 6)
encoding = ((0xEA + (Enc >> 8)) << 8) | (Enc & 0xFF)
constraints: Rd < 8, Rs < 8, Rb < 8
```

### `tadd Rd, Rs, #imm3`

If `Rd != Rs`:

```text
Lo = (Rs << 3) | Rd | ((imm3 & 0x3) << 6)
Hi = 0xEC + (imm3 >> 2)
encoding = (Hi << 8) | Lo
constraints: Rd < 8, Rs < 8, 0 <= imm3 <= 7
```

If `Rd == Rs`, use the `imm8` self-update form instead.

### `tsub Rd, Rs, #imm3`

If `Rd != Rs`:

```text
Enc = Rd | (Rs << 3) | (imm3 << 6)
encoding = ((0xEE + (Enc >> 8)) << 8) | (Enc & 0xFF)
constraints: Rd < 8, Rs < 8, 0 <= imm3 <= 7
```

If `Rd == Rs`, use the `imm8` self-update form instead.

### `tadd Rd, #imm8`

```text
encoding = ((0xB0 + Rd) << 8) | imm8
constraints: Rd < 8, 0 <= imm8 <= 255
```

### `tsub Rd, #imm8`

```text
encoding = ((0xB8 + Rd) << 8) | imm8
constraints: Rd < 8, 0 <= imm8 <= 255
```

These immediate self-update forms exist architecturally, but they are not safe for full-width pointer arithmetic.

## Two-Address Logical And Shift-By-Register Forms

All of the following forms use:

```text
encoding = Base | (Rs << 3) | Rd
constraints: Rd < 8, Rs < 8
```

| Instruction | Base |
| --- | --- |
| `tand Rd, Rs` | `0x0000` |
| `txor Rd, Rs` | `0x0040` |
| `tshftl Rd, Rs` | `0x0080` |
| `tshftr Rd, Rs` | `0x00C0` |
| `tasr Rd, Rs` | `0x0100` |
| `taddc Rd, Rs` | `0x0140` |
| `tsubc Rd, Rs` | `0x0180` |
| `trotr Rd, Rs` | `0x01C0` |
| `tor Rd, Rs` | `0x0300` |
| `tmul Rd, Rs` | `0x0340` |
| `tbclr Rd, Rs` | `0x0380` |
| `tmovn Rd, Rs` | `0x03C0` |

Printing rules:

- base `0x0380` prints as `tbclr`
- base `0x03C0` prints as the canonical TC32 unary-not form

## Shift-By-Immediate Forms

General formula:

```text
encoding = Base
         | ((EncImm >> 2) << 8)
         | ((EncImm & 0x3) << 6)
         | (Rs << 3)
         | Rd
constraints: Rd < 8, Rs < 8
```

Where:

- for left shift: `EncImm = imm`, `0 <= imm <= 31`
- for logical right shift: `EncImm = 0` encodes shift amount 32, otherwise `EncImm = imm`, `0 <= imm <= 32`
- for arithmetic right shift: `EncImm = 0` encodes shift amount 32, otherwise `EncImm = imm`, `0 <= imm <= 32`

Bases:

| Instruction | Base |
| --- | --- |
| `tshftl Rd, Rs, #imm` | `0xF000` |
| `tshftr Rd, Rs, #imm` | `0xF800` |
| `tasr Rd, Rs, #imm` | `0xE000` |

## Literal Load

### `tloadr Rt, [pc, #imm]`

```text
encoding = ((0x08 + Rt) << 8) | (imm >> 2)
constraints:
  Rt < 8
  imm is 4-byte aligned
  0 <= imm <= 1020
```

The effective address is:

```text
pc + 4 + imm
```

## PC-Relative Address Materialization

### `adr Rt, [pc, #imm]`

```text
encoding = 0xA000 | (Rt << 8) | (imm >> 2)
constraints:
  Rt < 8
  imm is 4-byte aligned
  0 <= imm <= 1020
```

The computed address is:

```text
pc + 4 + imm
```

## Word Load And Store With Immediate Offset

### Base register form

```text
word_offset = imm_bytes >> 2
encoding(load)  = 0x5800 | (word_offset << 6) | (Base << 3) | Rt
encoding(store) = 0x5000 | (word_offset << 6) | (Base << 3) | Rt
constraints:
  Rt < 8
  Base < 8
  imm_bytes is a multiple of 4
  0 <= imm_bytes <= 124
```

### Stack-pointer form

```text
encoding(load)  = ((0x38 + Rt) << 8) | word_offset
encoding(store) = ((0x30 + Rt) << 8) | word_offset
constraints:
  Rt < 8
  0 <= word_offset <= 255
  effective byte offset = word_offset << 2
```

## Byte Load And Store With Immediate Offset

```text
encoding(load)  = 0x4800 | (imm << 6) | (Base << 3) | Rt
encoding(store) = 0x4000 | (imm << 6) | (Base << 3) | Rt
constraints:
  Rt < 8
  Base < 8
  0 <= imm <= 31
```

## Halfword Load And Store With Immediate Offset

```text
half_offset = imm_bytes >> 1
encoding(load)  = 0x2800 | (half_offset << 6) | (Base << 3) | Rt
encoding(store) = 0x2000 | (half_offset << 6) | (Base << 3) | Rt
constraints:
  Rt < 8
  Base < 8
  imm_bytes is a multiple of 2
  0 <= imm_bytes <= 62
```

## Register-Offset Load And Store

General formula:

```text
encoding = BaseBits | (Off << 6) | (Base << 3) | Rt
constraints: Rt < 8, Base < 8, Off < 8
```

| Instruction | BaseBits |
| --- | --- |
| `tloadr Rt, [Base, Off]` | `0x1800` |
| `tstorer Rt, [Base, Off]` | `0x1000` |
| `tloadrb Rt, [Base, Off]` | `0x1C00` |
| `tstorerb Rt, [Base, Off]` | `0x1400` |
| `tloadrh Rt, [Base, Off]` | `0x1A00` |
| `tstorerh Rt, [Base, Off]` | `0x1200` |
| `tloadrsb Rt, [Base, Off]` | `0x1600` |
| `tloadrsh Rt, [Base, Off]` | `0x1E00` |

## Stack Pointer Adjustment

### `tadd sp, #imm`

```text
encoding = 0x6000 | imm_words
constraints:
  0 <= imm_words <= 127
  effective byte increment = imm_words << 2
```

### `tsub sp, #imm`

```text
encoding = 0x6080 | imm_words
constraints:
  0 <= imm_words <= 127
  effective byte decrement = imm_words << 2
```

## Add Immediate To Stack Pointer Into A Register

### `tadd Rt, sp, #imm`

```text
encoding = ((0x78 + Rt) << 8) | imm_words
constraints:
  Rt < 8
  0 <= imm_words <= 255
  effective byte offset = imm_words << 2
```

## Push And Pop

Low-register mask encoding:

- bit 0 corresponds to `r0`
- bit 1 corresponds to `r1`
- ...
- bit 7 corresponds to `r7`

### `tpush {reglist}`

```text
encoding = 0x6400 | mask
constraints:
  reglist may contain only r0-r7
```

### `tpush {reglist, lr}`

```text
encoding = 0x6500 | mask
constraints:
  reglist may contain r0-r7, and optionally lr
```

### `tpop {reglist}`

```text
encoding = 0x6C00 | mask
constraints:
  reglist may contain only r0-r7
```

### `tpop {reglist, pc}`

```text
encoding = 0x6D00 | mask
constraints:
  reglist may contain r0-r7, and optionally pc
```

## Multiple Load And Store

The first low-register operand is the base register. Remaining low registers set bits in the mask.

### `tloadm Base!, {reglist}`

```text
encoding = 0xD800 | (Base << 8) | mask
constraints:
  Base < 8
  reglist uses low registers
  mask does not encode the base register automatically
```

### `tstorem Base!, {reglist}`

```text
encoding = 0xD000 | (Base << 8) | mask
constraints:
  Base < 8
  reglist uses low registers
```

## Indirect Branch And Return

### `tjex Rm`

```text
encoding = 0x0700 | (Rm << 3)
constraints: Rm < 16
```

Examples:

```text
tjex lr -> 0x0770
tjex r1 -> 0x0708
```

## Short Conditional Branch

General formula:

```text
encoding = CondBase | (Enc & 0xFF)
where Enc = (target - 4) >> 1
constraints:
  target is 2-byte aligned
  Enc is signed 8-bit
```

Condition bases:

| Condition | Base |
| --- | --- |
| `EQ` | `0xC000` |
| `NE` | `0xC100` |
| `HS` | `0xC200` |
| `LO` | `0xC300` |
| `MI` | `0xC400` |
| `PL` | `0xC500` |
| `VS` | `0xC600` |
| `VC` | `0xC700` |
| `HI` | `0xC800` |
| `LS` | `0xC900` |
| `GE` | `0xCA00` |
| `LT` | `0xCB00` |
| `GT` | `0xCC00` |
| `LE` | `0xCD00` |

Special resolved-layout rule:

- `Enc = 0` means `target = P + 4`
- this is the case where the branch skips exactly one 16-bit instruction
- the 16-bit encoding is architecturally valid and decodable
- `must not generate`: the resolved 16-bit `Enc = 0` form on TLSR8258-class hardware
- `toolchain obligation`: rewrite that edge during final layout; if a compiler chooses local padding as the repair, one extra 16-bit filler instruction is sufficient

## Short Unconditional Branch

```text
Enc = (target - 4) >> 1
encoding = 0x8000 | (Enc & 0x07FF)
constraints:
  target is 2-byte aligned
  Enc is signed 11-bit
```

## Long Direct Branch

This encoding exists architecturally but is not safe as the default far-branch lowering.

```text
Enc    = (target - 4) >> 1
EncImm = Enc & 0x3FFFFF
encoding = 0x68009000
         | ((EncImm >> 11) & 0x07FF)
         | ((EncImm & 0x07FF) << 16)
constraints:
  target is 2-byte aligned
  Enc is signed 22-bit
```

## Long Direct Conditional Branch

This encoding exists architecturally but shall not be the default compiler lowering.

Compiler and assembler policy:

- `must not generate`: use this form as the general far-conditional strategy
- `must not generate`: synthesize this form as an automatic repair for a short conditional edge whose resolved target is exactly `P + 4`
- `decode requirement`: disassemblers and binary-analysis tools shall still recognize and decode this architectural form correctly when it appears in existing binaries or experiments

For condition code `CC`:

```text
Enc    = (target - 4) >> 1
EncImm = Enc & 0x7FFFF
Upper  = EncImm >> 11

FirstHalf  = 0x9000 | (CC << 6) | (Upper & 0x3F)
if Upper bit 7 is set: FirstHalf |= 0x0400

SecondHalf = 0x2000 | (EncImm & 0x07FF)
if Upper bit 6 is set: SecondHalf |= 0x5000
if Upper bit 7 is set: SecondHalf |= 0x0800

encoding = FirstHalf | (SecondHalf << 16)
constraints:
  target is 2-byte aligned
  Enc is signed 19-bit
```

Implementation note:

- the condition code bits belong to `FirstHalf`
- for little-endian TC32 object code, `FirstHalf` is the lower-addressed 16-bit halfword of the 32-bit instruction
- any assembler, relaxer, linker, or post-assembler fixup pass that rewrites an already-laid-out long conditional branch shall decode the preserved condition from the first halfword of the instruction itself
- if an implementation API already passes a byte pointer positioned at the fixup site, it must read `FirstHalf` from that pointer directly and must not add the instruction offset a second time
- otherwise a nonzero fragment-local instruction offset can corrupt the recovered condition and silently change, for example, `tjne` into `tjeq`

## Long Call Branch

### `tjl target`

```text
Enc    = (target - 4) >> 1
EncImm = Enc & 0x3FFFFF
encoding = 0x98009000
         | ((EncImm >> 11) & 0x07FF)
         | ((EncImm & 0x07FF) << 16)
constraints:
  target is 2-byte aligned
  Enc is signed 22-bit
```

## Decode Priority Rules

The decoder shall apply the following priorities:

1. decode literal-pool PC-relative load before generic shift/logical tables
2. decode `tjex` before generic high-register move/add/compare aliasing
3. distinguish `tshftr` from `tshftl` before alias printing
4. honor mapping symbols before attempting instruction decode in mixed code/data regions

## Architecturally Defined But Unsafe-To-Generate Forms

The following encodings are valid machine code but are not safe as the default output of an automatic compiler:

- direct long conditional branches
- the resolved 16-bit short conditional `Enc = 0` form
- direct long `tj`
- short-immediate pointer arithmetic using `tadd #imm` or `tsub #imm`

These forms may still appear in hand-written assembly, diagnostic tools, or controlled experiments. The one exception for automatic code generation is the mandatory widening of a resolved short conditional edge whose target is exactly `P + 4`.
