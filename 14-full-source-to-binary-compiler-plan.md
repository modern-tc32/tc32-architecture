# 14. Full Source-To-Binary Compiler Plan

## Purpose

This chapter answers a stricter question than the rest of the specification:

```text
What is required to build a completely new compiler toolchain that turns
high-level source code into a bootable TC32 binary image?
```

It remains language-agnostic. It does not define the syntax or semantics of any particular source language.
It does define the complete target-facing contract that a new C-like or Rust-like compiler must obey.

## Sufficiency Assessment

The document set in this directory is sufficient for the TC32-specific half of a full compiler toolchain if, and only if, the implementation also chooses its own source-language definition.

Covered completely after this chapter:

- instruction encodings
- branch formulas and safe/unsafe branch-selection rules
- register file, flags, and condition evaluation
- calling convention and stack discipline
- object layout and relocation semantics
- linked image and flat-binary rules
- startup and reset contract
- data-layout profile for target-visible storage
- runtime-helper ABI contract
- legalization policy for operations not directly supported by the base ISA
- implementation order for a new source-to-binary toolchain

Still intentionally outside the scope of this directory:

- source-language grammar
- parser error-recovery rules
- type-system rules of a specific language
- name resolution rules of a specific language
- macro expansion model
- package manager behavior
- borrow checking, trait solving, or template instantiation
- source-language standard library API surface

Conclusion:

- `yes`: this directory is now sufficient for implementing the TC32 target side of a full `source -> obj -> elf -> bin` compiler pipeline from scratch
- `no`: this directory alone does not define the whole C language or the whole Rust language, because that would violate the language-agnostic boundary

## Target-Visible Data Layout Profile

Any front-end that wants interoperable object files and linked images shall lower target-visible data using the following storage profile.

### Scalar Storage Units

| Kind | Size | Alignment |
| --- | --- | --- |
| 8-bit integer | 1 byte | 1 byte |
| 16-bit integer | 2 bytes | 2 bytes |
| 32-bit integer | 4 bytes | 4 bytes |
| 64-bit integer | 8 bytes | 4 bytes |
| data pointer | 4 bytes | 4 bytes |
| code pointer | 4 bytes | 4 bytes |
| boolean in memory | 1 byte | 1 byte |

Rules:

- memory is little-endian
- one byte is 8 bits
- a code pointer stored in memory is `function_address + 1`
- a null code pointer is the all-zero word
- a boolean stored in memory shall be `0` for false or `1` for true
- a boolean held in a register shall use the 32-bit values `0` or `1`

### Aggregate Layout

Language-neutral aggregate layout rules for externally visible binary interfaces:

- each field is placed at the next offset satisfying that field's alignment
- aggregate alignment is the maximum field alignment, capped at 4
- aggregate size is rounded up to aggregate alignment
- an empty aggregate, if admitted by the source language, occupies 0 bytes internally but shall not be used directly as a binary interface value

Example:

```text
struct-like aggregate:
  field a: u8   at offset 0
  field b: u32  at offset 4
  field c: u16  at offset 8

aggregate alignment = 4
aggregate size      = 12
```

### Register Representation Of Narrow Values

When a sub-32-bit integer value is held in a register:

- the value occupies the low bits of the 32-bit register
- high bits are either sign-extended or zero-extended according to the operation that produced the value
- before storing an 8-bit or 16-bit object to memory, the compiler shall emit a byte or halfword store or otherwise explicitly truncate the value

## Front-End Output Contract

Before the TC32 backend sees the program, the frontend or a target-neutral middle-end shall normalize it into a form with the following properties:

- all control flow is explicit
- all loads and stores carry size and alignment information
- all by-value aggregate transfers are explicit copy operations
- all code and data addresses are explicit values
- all indirect calls name the function-pointer operand explicitly
- all operations wider than 32 bits are explicit in the IR
- all helper-requiring operations are either still present as explicit abstract operations or already lowered to helper calls

The IR must not rely on:

- implicit trap instructions
- implicit atomics
- target-unknown packed-struct access behavior
- hidden constructor or destructor invocation
- a generic "call any pointer" primitive without target lowering rules

## Canonical Result And Booleanization Rules

The backend shall use the following canonical result rules:

- integer comparison used only by a branch may stay in flags and branch directly
- integer comparison materialized as a value shall produce register `0` for false and `1` for true
- logical negation of a boolean value shall preserve the `0` or `1` representation
- a nonzero-test lowered into a value shall compare against zero and materialize `0` or `1`

Safe value materialization shape:

```asm
tcmp r0, #0
tjne .Ltrue
tmov r0, #0
tj   .Ldone
.Ltrue:
tmov r0, #1
.Ldone:
```

`safe to generate`: prefer branch-based booleanization over condition-move designs.

## Indirect Calls And Code Pointers

The canonical indirect-call design is:

```asm
; arguments already placed in r0-r3 and on the stack
; target pointer held in any register, here r2
tmov r12, r2
tjl  __tc32_indirect_call_r12
```

Helper body:

```asm
__tc32_indirect_call_r12:
    tjex r12
```

Properties:

- `tjl` creates the correct return linkage in `lr`
- the helper preserves ordinary call semantics
- the callee returns directly to the original caller continuation
- argument registers remain available to the callee

`must not generate`: ad hoc indirect-call sequences that fabricate a return address from raw `pc` arithmetic unless the implementation proves them equivalent to the veneer above.

## Legalization Matrix

This section defines the minimum complete legalization policy for a fresh compiler.

### 8-Bit And 16-Bit Integer Arithmetic

Use 32-bit registers for computation.

Rules:

- add, subtract, and, or, xor, compare, and shifts may be computed in 32-bit registers
- when the source-language operation wraps at 8 or 16 bits, truncate or store back through the correct narrow operation
- sign extension shall use arithmetic shifts or sign-extending loads
- zero extension shall use logical masking or zero-extending loads

Safe sign-extension examples:

```asm
; sign-extend low byte of r0
tshftl r0, r0, #24
tasr   r0, r0, #24
```

```asm
; sign-extend low halfword of r0
tshftl r0, r0, #16
tasr   r0, r0, #16
```

### 32-Bit Integer Arithmetic

Directly express with native operations:

- add
- subtract
- compare
- logical operations
- shifts
- rotate right
- multiply

Required restrictions:

- use explicit compare before any branch that depends on precise signed or unsigned ordering when the preceding value-producing instruction is not itself the intended compare
- do not use short-immediate pointer arithmetic forms for full-width pointer values

### 64-Bit Integer Arithmetic

A complete compiler may either inline or helper-lower 64-bit arithmetic.

Minimum required support:

- add
- subtract
- compare
- left shift
- logical right shift
- arithmetic right shift
- multiply
- divide
- remainder

Recommended minimum implementation policy:

- inline 64-bit add and subtract
- inline simple equality and inequality comparison
- helper-lower 64-bit multiply, divide, remainder, and variable shifts for the first working compiler

Safe inline 64-bit add example:

```asm
; (r0:r1) = (r0:r1) + (r2:r3)
tadd  r0, r2
taddc r1, r3
```

Safe inline 64-bit subtract example:

```asm
; (r0:r1) = (r0:r1) - (r2:r3)
tsub  r0, r2
tsubc r1, r3
```

Safe 64-bit equality test outline:

```text
if high_word_a != high_word_b: false
else if low_word_a != low_word_b: false
else true
```

### Division And Remainder

The first complete compiler should treat division and remainder as helper operations.

Required semantic cases:

- unsigned 32-bit divide
- signed 32-bit divide
- unsigned 32-bit remainder
- signed 32-bit remainder
- if the source language admits 64-bit integers, the same four operations at 64-bit width

Division-by-zero behavior is source-language-defined.
The backend contract is:

- if the source language requires a trap, call a no-return trap helper or emit an equivalent non-returning path
- if the source language leaves the result undefined, the compiler may leave the case unchecked

### Floating Point

This specification defines no hardware floating point.

Therefore:

- `must not generate`: direct hardware floating-point instructions
- `safe to generate`: helper calls implementing the chosen floating-point model

The language implementation must choose:

- software IEEE-like semantics
- reduced semantics
- or a front-end-level prohibition on floating-point support

### Memory Operations

The compiler shall support:

- byte load and store
- halfword load and store
- word load and store
- block copy
- block move
- block fill

Rules:

- aligned byte, halfword, and word operations may use direct scalar instructions
- misaligned halfword and word accesses shall be expanded into safe byte operations
- large aggregate copies may become helper calls
- small aggregate copies may inline as byte or word moves

### Switch Lowering

A complete compiler shall support both:

- branch-chain lowering
- jump-table lowering

Safe policy:

- for sparse switches, use ordered compare-and-branch chains
- for dense switches, use the inline 32-bit jump-table format defined earlier in this specification
- use `tjhi` rather than `tjls` for upper-bound checks

### Select And Ternary Operations

Because the minimal TC32 profile does not require a general conditional-move instruction surface:

- `safe to generate`: lower select-like operations into CFG branches and block merges
- `safe to generate`: materialize boolean results with explicit branch structure

## Runtime Helper ABI Contract

All helpers use the ordinary TC32 calling convention.
The helper symbol names below are recommended canonical names for a fresh toolchain.
An implementation may rename them internally, but the signatures and observable semantics shall remain equivalent.

### Memory Helpers

```text
__tc32_memcpy(dst: r0, src: r1, len: r2) -> r0 = dst
__tc32_memmove(dst: r0, src: r1, len: r2) -> r0 = dst
__tc32_memset(dst: r0, byte_value: r1, len: r2) -> r0 = dst
__tc32_bzero(dst: r0, len: r1) -> no meaningful result
```

### 32-Bit Integer Helpers

```text
__tc32_udiv32(num: r0, den: r1) -> r0
__tc32_sdiv32(num: r0, den: r1) -> r0
__tc32_urem32(num: r0, den: r1) -> r0
__tc32_srem32(num: r0, den: r1) -> r0
```

### 64-Bit Integer Helpers

Argument packing:

- first 64-bit value in `r0:r1`
- second 64-bit value in `r2:r3`
- additional arguments on the stack

Return packing:

- 64-bit result in `r0:r1`

Recommended helper set:

```text
__tc32_add64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_sub64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_mul64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_udiv64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_sdiv64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_urem64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_srem64(lo_a: r0, hi_a: r1, lo_b: r2, hi_b: r3) -> r0:r1
__tc32_shl64(lo: r0, hi: r1, sh: r2) -> r0:r1
__tc32_lshr64(lo: r0, hi: r1, sh: r2) -> r0:r1
__tc32_ashr64(lo: r0, hi: r1, sh: r2) -> r0:r1
```

### Floating-Point Helpers

If floating point is admitted, the implementation shall define a stable software-helper ABI for:

- add
- subtract
- multiply
- divide
- conversion between integer and floating-point values
- comparison

The exact bit-level floating format is a language-level choice, but the helper ABI must be internally consistent across all compilation units.

### Trap And Abort Helpers

Recommended no-return helpers:

```text
__tc32_abort() -> no return
__tc32_trap_divzero() -> no return
__tc32_unreachable() -> no return
```

## Startup Hooks For Real Frontends

For a source language that supports global initialization, the runtime shall support:

- `.preinit_array`
- `.init_array`
- optional `.fini_array`

Execution order:

1. copy initialized writable data
2. zero zero-initialized writable data
3. run `.preinit_array`
4. run `.init_array`
5. call the ordinary program entry routine
6. optionally run `.fini_array` only on an orderly exit path

Constructor arrays store ordinary TC32 code pointers:

```text
stored entry = callback_address + 1
```

## Translation-Unit And Linkage Rules

For multi-file compilation:

- every externally callable function symbol shall denote the aligned code entry address
- every stored function pointer relocation shall write `symbol + 1`
- every stored data pointer relocation shall write `symbol`
- helper symbols shall be ordinary global function symbols
- constructor arrays shall contain relocatable code-pointer words

The linker shall remain free to place functions in any order that still satisfies branch and thunk rules.

## Recommended End-To-End Implementation Order

The most reliable order for a brand-new compiler is:

1. define the target-visible data layout
2. implement the instruction encoder
3. implement the instruction decoder and disassembler
4. implement the assembler
5. implement the relocatable object writer
6. implement the linker
7. implement the flat-binary emitter
8. implement reset/startup runtime and helper library
9. implement a tiny target-neutral IR
10. implement straight-line 32-bit code generation
11. implement calls, returns, and stack frames
12. implement indirect-call veneers
13. implement conditional branches and branch rewrites
14. implement switch lowering and jump tables
15. implement hazard repair
16. implement 64-bit legalization and helpers
17. implement source-language frontends
18. implement optimizer passes only after correctness is stable

This order matters.
If parsing and semantic analysis are built first, the project can accumulate high-level features while still lacking a trustworthy path to machine code.

## Full Compiler Plan

### Phase A: Machine Core

Deliver:

- encoder
- decoder
- disassembler
- assembler diagnostics

Exit condition:

- every instruction in the complete encoding reference round-trips through encode and decode

### Phase B: Object And Image Toolchain

Deliver:

- section writer
- symbol table writer
- relocation writer
- ELF linker
- flat-binary extractor

Exit condition:

- a hand-written startup file and one hand-written function can link into a bootable image

### Phase C: Runtime Core

Deliver:

- reset entry
- copy and zero loops
- optional init-array walker
- indirect-call veneer
- memory helpers
- division helpers

Exit condition:

- a linked image can initialize memory and call a normal function entry point

### Phase D: Minimal Backend

Deliver:

- integer constants
- integer arithmetic
- loads and stores
- direct calls
- returns
- stack frames

Exit condition:

- straight-line scalar functions compile correctly

### Phase E: Full Control Flow

Deliver:

- compare lowering
- conditional branches
- branch repair
- far-edge veneers
- switch lowering
- jump tables

Exit condition:

- loops, if-else chains, and dense switches compile correctly without forbidden branch patterns

### Phase F: Full Scalar Support

Deliver:

- 8-bit and 16-bit normalization
- 64-bit add/sub support
- helper-lowered wide operations
- varargs support
- aggregate passing support

Exit condition:

- the backend can support the scalar and aggregate core of a systems language

### Phase G: Frontend Integration

Deliver:

- parser
- semantic analysis
- target-neutral IR
- lowering to TC32 backend

Exit condition:

- multi-file source programs compile through to a bootable binary with no hand-written assembly

## Minimum Conformance Program Set

A fresh compiler should not be considered complete until it can compile and run programs that require:

- arithmetic on 8, 16, 32, and 64-bit integers
- signed and unsigned comparison
- local stack objects
- direct calls
- indirect calls through function pointers
- global initialized data
- zero-initialized data
- constructor arrays if the source language supports global initialization
- loops
- recursion
- sparse switches
- dense switches
- structure passing by value
- variadic access if the source language admits it
- helper-lowered division and remainder
- helper-lowered memory copy and fill

## Final Practical Recommendation

If the goal is a completely new compiler rather than a thin wrapper around an existing implementation:

- keep one small target-neutral IR
- keep one strict TC32 backend that owns all machine-sensitive rules
- keep assembler, linker, and runtime independent from the frontend
- standardize helper ABIs early
- add optimization only after the unoptimized compiler can already produce correct binaries

That architecture allows one TC32 target implementation to serve both a C-like frontend and a Rust-like frontend without duplicating the hard TC32-specific logic.
