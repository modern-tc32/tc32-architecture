# 07. Safe And Unsafe Code-Generation Patterns

This chapter is the normative summary for miscompile-prone cases.

## Pattern Classification

Each pattern is classified as:

- `safe to generate`
- `must not generate`

The existence of an encoding does not make it safe.

## Pointer Arithmetic

Safe:

```asm
tmov r1, #1
tadd r0, r1
```

Unsafe:

```asm
tadd r0, #1
```

## Conditional Far Branch

Safe:

```asm
tjne .Lskip
tjl  far_target
.Lskip:
```

Unsafe:

```asm
; direct long conditional branch shape
tj<cc>.long far_target
```

## Signed Nonnegative Test

Safe:

```asm
tcmp r0, #0
tjlt .Lnegative
tj   .Lnonnegative
```

Unsafe:

```asm
tjpl .Lnonnegative
```

## Jump-Table Range Check

Safe:

```asm
tcmp r0, #7
tjhi .Ldefault
```

Unsafe:

```asm
tcmp r0, #7
tjls .Ldispatch
```

## Indirect Dispatch Table Entry

Safe:

```asm
.long case_label + 1
```

Unsafe:

```asm
.long case_label
```

## Large-Frame Return

Safe:

```asm
tadd sp, #frame_size
tpop {r4, r5, r6, r7, pc}
```

Unsafe:

```asm
tpop {r3, r4, r5, r6, r7, pc}
```

## Saved-Argument-Area Return

Safe:

```asm
tpop   {r1}
nop
nop
tadd   sp, #saved_argument_area_size
tjex   r1
```

Unsafe:

```asm
tloadr r1, [sp]
tstorer r1, [sp, #saved_argument_area_size]
tadd   sp, #saved_argument_area_size
tpop   {pc}
```

For 64-bit returns in `r0:r1`, the return-address temporary must not be `r0` or
`r1`. Use another low register and apply the normal post-`tpop` dependency
padding before consuming it.

## Load Dependency

Safe:

```asm
tloadr r0, [r1, #0]
nop
nop
tcmp   r0, #0
```

Unsafe:

```asm
tloadr r0, [r1, #0]
tcmp   r0, #0
```

## Pop Dependency

Safe:

```asm
tpop {r0}
nop
nop
tcmp r0, #0
```

Unsafe:

```asm
tpop {r0}
tcmp r0, #0
```

## Literal Load Beyond Range

Safe:

- use an in-range literal slot
- or synthesize the address through a helper sequence chosen by code generation

Unsafe:

```asm
tloadr r0, [pc, #literal_out_of_range]
```

## Misaligned Access

Safe:

- byte-wise expansion
- helper routine

Unsafe:

- direct halfword or word access to an address not aligned for that access width

## Unsupported Atomic Operation

Safe:

- plain load
- plain store

Unsafe:

- compare-and-swap
- exclusive monitor sequences
- multithreaded atomic read-modify-write operations

## General Rule

Whenever a pattern in this chapter is marked unsafe, the compiler shall rewrite the program into one of the safe patterns before assembly is finalized.
