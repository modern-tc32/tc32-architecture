# 11. Worked Examples And Conformance Scenarios

This chapter gives inline examples that a fresh TC32 implementation should be able to encode, link, and execute correctly.

## Example 1: Maximum Forward Short Conditional Branch

Instruction at `0x00000000`:

```asm
tjeq target
```

Target at `0x00000102` is valid because:

```text
base        = 0x00000004
displacement= 0x00000102 - 0x00000004 = 0x000000fe
```

`0x00fe` is within the short conditional range.

## Example 2: Maximum Forward Short Unconditional Branch

Instruction at `0x00000000`:

```asm
tj target
```

Target at `0x00000802` is valid because:

```text
base        = 0x00000004
displacement= 0x00000802 - 0x00000004 = 0x000007fe
```

`0x07fe` is within the short unconditional range.

## Example 3: Maximum Forward Long Call

Instruction at `0x00000000`:

```asm
tjl target
```

Target at `0x00400002` is valid because:

```text
base        = 0x00000004
displacement= 0x00400002 - 0x00000004 = 0x003ffffe
```

`0x003ffffe` is within the `tjl` range.

## Example 4: Maximum Literal-Load Reach

Instruction at `0x00000000`:

```asm
tloadr r0, [pc, #0x3f8]
```

Effective address:

```text
literal = 0x00000004 + 0x000003f8 = 0x000003fc
```

So the literal word may sit at `0x000003fc`.

## Example 5: Safe Pointer Increment

Correct:

```asm
tmov r1, #1
tadd r0, r1
```

Incorrect:

```asm
tadd r0, #1
```

## Example 6: Safe Far Conditional Transfer

Correct:

```asm
tjne .Lskip
tjl  far_target
.Lskip:
```

Incorrect:

```asm
; any direct long conditional form
```

## Example 6A: Adjacent Conditional Branch Safety Repair

Source shape:

```asm
tjne .Ltarget
nop
.Ltarget:
```

Interpretation:

- the resolved target is exactly `P + 4`
- a literal 16-bit `tjne` encoding would therefore use displacement `0`
- that resolved short form is not safe as generated output on TLSR8258-class hardware

Required result:

- the backend shall rewrite the edge before final assembly
- acceptable repairs include local CFG reshaping or adding one 16-bit filler instruction so the target is no longer exactly `P + 4`
- the assembler or MC relaxer shall not silently turn this case into a direct long conditional branch

## Example 7: Safe Jump Table

```asm
tcmp   r0, #3
tjhi   .Ldefault
tshftl r0, r0, #2
tadd   r0, pc
tloadr r0, [r0, #8]
nop
nop
tmov   pc, r0
.align 2
.long .Lcase0 + 1
.long .Lcase1 + 1
.long .Lcase2 + 1
.long .Lcase3 + 1
```

Properties:

- uses `tjhi` instead of `tjls`
- uses 32-bit entries
- stores code pointers with bit 0 set
- includes the required two `nop` after the table load

## Example 8: Safe Saved-Return-Slot Epilogue

```asm
tloadr  r0, [sp, #saved_ra]
tstorer r0, [sp, #0]
tadd    sp, #frame_size
tpop    {pc}
```

This is the canonical shape when the return address must be moved back to the top stack slot before returning.

## Example 9: Canonical Long-Call Thunk

```asm
tpush   {r0, r1}
tloadr  r0, [pc, #16]
nop
nop
tstorer r0, [sp, #4]
nop
nop
tpop    {r0, pc}
nop
nop
.word absolute_target
```

Any linker range-extension implementation shall preserve these ordering constraints and hazard pads.

## Example 10: Minimal Reset Path

```asm
__start:
    b __reset

__reset:
    ldr r0, =_stack_end_
    mov sp, r0
    tjl __tc32_boot_init

__tc32_boot_init:
    ; initialize writable memory
    tjl main_entry
.Lhang:
    tj .Lhang
```

This example is sufficient to describe the minimum control-flow skeleton of a bootable image.

## Conformance Summary

A TC32 implementation is structurally complete when it can:

- encode and decode the branch and literal examples above
- preserve function-pointer low-bit semantics
- generate only the safe patterns in this chapter
- link out-of-range calls through the canonical thunk
- build a flash image beginning at address zero
- initialize SRAM before transferring control to the program entry routine
