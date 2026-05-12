# 01. CPU And Execution Model

## Core Machine

- TC32 is a 32-bit little-endian machine.
- The architectural register file contains `r0` through `r15`.
- `r13` is `sp`, `r14` is `lr`, and `r15` is `pc`.
- Instruction fetch is organized in 16-bit units.
- Instructions are either:
  - one 16-bit halfword, or
  - one 32-bit instruction formed from two adjacent 16-bit halfwords.

## Code State And Address Representation

- TC32 executes only TC32 code in this specification.
- A code address stored in data memory shall use bit 0 set to `1`.
- A data address stored in memory shall use bit 0 according to its natural numeric value.
- `toolchain obligation`: when emitting function pointers, jump-table targets, callback tables, or any other stored code address, store `symbol_address + 1`.
- `toolchain obligation`: when executing an indirect control transfer through a register, the machine-visible code pointer value may carry bit 0 set, but the fetched instruction stream starts at the aligned code address.

Example:

```text
stored function pointer = 0x00001235
actual code entry       = 0x00001234
```

## Register Use Model

- `r0` to `r3` are the primary argument, result, and scratch registers.
- `r4` to `r7` are the primary callee-saved low registers.
- `r12` is a caller-saved interprocedural scratch register and is the preferred register for indirect-call veneers.
- TC32 data-processing and memory instructions strongly favor low registers `r0` to `r7`.
- High-register forms exist for at least selected operations involving `r8` and `lr`.
- `safe to generate`: prefer low-register allocation whenever a choice exists.

## Program Counter Rules

- PC-relative control transfers and literal loads use a `pc + 4` bias.
- The branch or literal computation base is the address of the current instruction plus `4`.
- The bias rule applies to:
  - short unconditional branches
  - short conditional branches
  - long call branches
  - PC-relative literal loads
  - inline jump-table dispatch

Examples:

```text
instruction address  = 0x00000000
branch base          = 0x00000004
literal-load base    = 0x00000004
```

## Condition State

TC32 exposes the standard condition state bits:

- `N`: negative
- `Z`: zero
- `C`: carry / not-borrow
- `V`: signed overflow

Flag meaning:

- `N = 1` when the flag-producing result has bit 31 set.
- `Z = 1` when the flag-producing result is exactly zero.
- `C = 1` after unsigned addition when the operation produces a carry out of bit 31.
- `C = 1` after unsigned subtraction or compare when no borrow is required.
- `C` after shifts and rotates is the last bit shifted out when the shift amount is nonzero.
- `V = 1` when a signed add, subtract, compare, add-with-carry, subtract-with-carry, or compare-negative operation overflows in two's-complement arithmetic.

Persistence rule:

- condition flags remain live until the next instruction that overwrites them
- instructions that do not define flags leave the previous condition state intact

These flags are consumed by short conditional branches.

## Condition Evaluation Rules

For every short conditional branch `tj<cc>`, the branch decision is:

| Condition | Meaning | Boolean rule |
| --- | --- | --- |
| `EQ` | equal | `Z == 1` |
| `NE` | not equal | `Z == 0` |
| `HS` | unsigned higher or same | `C == 1` |
| `LO` | unsigned lower | `C == 0` |
| `MI` | negative | `N == 1` |
| `PL` | positive or zero | `N == 0` |
| `VS` | overflow | `V == 1` |
| `VC` | no overflow | `V == 0` |
| `HI` | unsigned higher | `C == 1 and Z == 0` |
| `LS` | unsigned lower or same | `C == 0 or Z == 1` |
| `GE` | signed greater or equal | `N == V` |
| `LT` | signed less than | `N != V` |
| `GT` | signed greater than | `Z == 0 and N == V` |
| `LE` | signed less than or equal | `Z == 1 or N != V` |

The table above is the architectural meaning of the condition decoder.

## Reliable And Unreliable Flag Consumers

Reliable branch-source rules:

- `safe to generate`: use `EQ` and `NE` after compare, add, subtract, add-with-carry, subtract-with-carry, logical-result, move-result, multiply-result, and shift-result instructions whose result is intentionally being tested for zero.
- `safe to generate`: use `MI` after compare, add, subtract, add-with-carry, subtract-with-carry, move-result, or shift-result instructions when the sign bit of the produced result is intentionally being tested.
- `safe to generate`: use `HS`, `LO`, `HI`, `LT`, `GT`, and `LE` only when the immediately preceding flag source is an explicit compare or arithmetic operation whose unsigned-carry or signed-overflow meaning is part of the intended test.
- `must not generate`: rely on stale flags across helper calls, indirect transfers, or any other instruction sequence that may insert an intervening flag writer.
- `must not generate`: rely on carry-sensitive conditions after a variable shift unless that exact carry-out behavior is intentionally required and validated.

## Condition-Code Reliability

The following condition codes are `architecturally defined` but require conservative lowering rules:

- `GE`
- `PL`
- `LS`

Rules:

- `safe to generate`: short conditional branches using ordinary conditions such as `EQ`, `NE`, `HS`, `LO`, `MI`, `HI`, `LT`, `GT`, and `LE`, provided the branch is in range and the resolved target is not exactly `P + 4`.
- `must not generate`: the resolved 16-bit `tj<cc>` encoding whose target is exactly `P + 4`, because that encoding corresponds to `Enc = 0` and misexecutes on TLSR8258-class hardware.
- `must not generate`: direct long conditional branches as the general far-edge lowering strategy.
- `toolchain obligation`: when a resolved short conditional edge targets exactly `P + 4`, rewrite or widen that edge during final layout instead of emitting the 16-bit `Enc = 0` form.
- `must not generate`: `tjpl` as the primary lowering of signed `>= 0` after an arbitrary callback or helper return.
- `must not generate`: `tjls` as the primary range-check branch for jump-table dispatch.

Preferred safe shapes:

- For signed nonnegative tests, generate an explicit compare and branch on the opposite condition:

```asm
tcmp  r0, #0
tjlt  .Lnegative
tj    .Lnonnegative
```

- For single-bit tests, prefer shift-and-sign-branch form:

```asm
tshftl r0, r0, #31
tjmi   .Lbit_set
```

Practical source rules:

- `safe to generate`: `tcmp` or `tcmpn` immediately followed by the consuming branch.
- `safe to generate`: `tcmp value, #0` before signed or unsigned range checks.
- `safe to generate`: `tshftl value, value, #31` followed by `tjmi` for top-bit tests.
- `must not generate`: `tjpl` as a shorthand for "last result was nonnegative" when the last flag writer is not an explicit local compare.
- `must not generate`: `tjls` as a shorthand for switch upper-bound checks.
- `must not generate`: `tjge` as the primary lowering when an equivalent explicit compare plus `tjlt`/fallthrough form exists.

## Stack Model

- The stack grows toward lower addresses.
- The stack pointer always refers to byte-addressed memory.
- All ordinary stack slots in this specification are word-sized and word-aligned.
- `safe to generate`: keep `sp` 4-byte aligned at every call boundary.
- `toolchain obligation`: frame setup and teardown must preserve word alignment across calls, spills, and returns.

## Return Model

Not every PC-restoring encoding is safe for automatically generated returns.

Safe return forms:

- `tpop {r4, r5, r6, r7, pc}`
- `tpop {pc}`
- `tjex lr` when no stack-restoring return is required

Unsafe return forms:

- `tpop {r3, r4, r5, r6, r7, pc}`
- split returns that restore a saved return address into a scratch register and then jump through that scratch register after additional stack adjustment

Example of a shape that `must not generate`:

```asm
tpop {r4, r5, r6, r7}
tpop {r1}
tadd sp, #16
tjex r1
```

## Special Instructions

The core special instruction set used by TC32 toolchains is:

- `tmcsr`
- `tmssr`
- `tmrss`
- `treti`
- `nop`

Semantics:

- `tmcsr`: writes control or status state from a general register.
- `tmssr`: writes the interrupt-mask/status register used by TC32 privileged control flow.
- `tmrss`: reads that register into a general register.
- `treti`: returns from interrupt context.
- `nop`: performs no architecturally visible state change.

## Floating-Point And Vector Model

This specification defines no hardware floating-point register file and no hardware vector execution model.

Rules:

- floating-point operations, if admitted by a front-end, shall lower through software helper routines
- vector operations, if admitted by a front-end, shall lower through scalar code or helper routines

## Unsupported System Instruction Surface

The following instruction families are outside the TC32 code-generation surface defined here and shall be rejected by a TC32 assembler:

- breakpoint instructions
- exclusive-access instructions
- generic ARM interrupt-enable and interrupt-disable mnemonics
- wait-for-event and wait-for-interrupt instructions
- barrier instructions
- supervisor, monitor, and hypervisor call instructions
- undefined-instruction pseudo-forms

## Reliability Summary

The following forms are `architecturally defined` but not safe as primary code-generation targets for TLSR8258-class hardware:

- direct long conditional branches, except when used only as the mandatory repair for a short conditional edge whose resolved target is exactly `P + 4`
- unchecked far direct `tj`
- condition-code-sensitive shapes relying on `PL`, `GE`, or `LS` when a safer short-branch rewrite exists

When such a form would otherwise be selected, the toolchain shall rewrite the control flow into a safe short-branch-plus-veneer shape.
