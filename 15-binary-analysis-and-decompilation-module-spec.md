# 15. Binary Analysis And Decompilation Module Specification

## Purpose

This chapter defines the target-facing contract for any tool that:

- disassembles TC32 machine code
- reconstructs control flow from TC32 binaries
- lifts instructions into an analysis IR
- performs stack, function, and switch recovery
- decompiles TC32 code into a higher-level representation

It is intentionally product-agnostic.
It does not prescribe one specific internal IR or one specific user interface.
It does define the observable architectural facts and recovery rules that such a tool shall use.

## Sufficiency Statement

With this chapter added, the document set is sufficient for implementing a TC32 binary-analysis or decompilation module from scratch.

That means the documentation now covers:

- register and flag semantics
- exact instruction encodings
- code and data pointer representation
- relocation meaning
- section and memory-region meaning
- startup and interrupt entry structure
- branch, thunk, and jump-table recovery rules
- stack and frame conventions
- safe and unsafe instruction patterns that affect analysis

What remains implementation-defined:

- the internal analysis IR
- naming heuristics for unnamed local variables
- presentation of decompiled syntax
- user-interface features
- debug-info integration beyond the object-format facts already described here

## Analysis Input Model

A TC32 analysis module shall accept any of the following inputs:

- raw flash binary image
- relocatable object file
- linked ELF image

The preferred analysis source is a linked ELF image because it may preserve:

- section boundaries
- relocation records
- mapping symbols
- symbol names

If only a raw flash binary is available, the tool shall still be able to analyze code by using:

- the canonical flash base address `0x00000000`
- the boot header layout
- the reset and interrupt entry rules
- the instruction encoding reference

## Architectural State Model

The minimal analysis-visible architectural state is:

- general registers `r0` through `r15`
- aliases `sp = r13`, `lr = r14`, `pc = r15`
- condition flags `N`, `Z`, `C`, `V`
- byte-addressed little-endian memory

The analysis module shall model:

- every register as 32 bits
- every ordinary memory address as a byte address
- every code pointer stored in memory as `code_address + 1`
- every data pointer stored in memory as the raw aligned data address

## Code And Data Region Model

The tool shall distinguish at least the following region classes:

- executable flash code
- read-only embedded data inside code sections
- writable SRAM data
- zero-initialized SRAM data
- vector and boot-header region
- early-availability code region
- constructor and initialization callback arrays
- stack region
- MMIO region

Recommended base memory interpretation:

- flash execution base: `0x00000000`
- SRAM base: `0x00840000`
- principal MMIO window around `0x00800000`

The tool shall treat accesses into the MMIO window as memory-mapped side effects, not as ordinary RAM aliasing.

## Mapping-Symbol And Section Rules

If mapping symbols are present:

- `$t` starts executable instruction bytes
- `$d` starts embedded data bytes
- `$d.rodata` starts embedded read-only data bytes

Rules:

- instruction decoding shall stop at the first byte marked as data
- embedded words in `$d` and `$d.rodata` regions shall not be speculatively decoded as instructions
- if relocations indicate an absolute word points to code, the displayed value may still be shown as a code pointer even inside a data region

If mapping symbols are absent:

- section permissions, section names, and control-flow reachability shall drive the initial code/data split
- the tool may speculate additional function starts only from valid control-flow targets or known startup entry points

## Instruction Decode Contract

The decoder shall use the exact encodings and priorities defined elsewhere in this specification.

Mandatory decoder behaviors:

- respect 16-bit and 32-bit instruction widths exactly
- use the `pc + 4` bias for branch and literal computations
- decode `tshftr` before generic shift aliases
- decode `tjex` before generic high-register move/add/compare aliasing
- preserve the canonical instruction names used by this specification

If a byte stream cannot be decoded as a valid instruction:

- mark it as undecodable data or invalid instruction bytes
- do not silently reinterpret it through a different ISA mode

## Lifted Instruction Semantics

For analysis and decompilation, each instruction shall be interpreted as a set of abstract actions over:

- register reads
- register writes
- memory reads
- memory writes
- flag reads
- flag writes
- control-flow edges

### Core Semantic Categories

The lifted instruction model shall distinguish at least:

- pure data movement
- arithmetic with full flag effects
- logical operations with partial flag effects
- shift and rotate operations with carry propagation
- compare-only operations with no general-register result
- loads and stores
- direct branches
- conditional branches
- direct calls
- indirect transfers
- interrupt return
- special-register transfer

### Control-Flow Classification

Each instruction shall be tagged as one of:

- ordinary fallthrough instruction
- conditional branch
- unconditional branch
- direct call
- indirect jump
- indirect return
- interrupt return
- non-returning instruction if known by context rather than opcode

Classification rules:

- `tj` is an unconditional direct branch
- `tj<cc>` is a conditional direct branch with fallthrough
- `tjl` is a direct call
- `tjex lr` is an ordinary return candidate
- `tjex reg` for `reg != lr` is an indirect jump candidate
- `treti` is an interrupt-return instruction

## Function-Start Recovery

The analysis module shall recognize function starts from the following evidence sources.

### Strong Evidence

- exported or local symbols that denote code addresses
- direct-call targets of `tjl`
- stored code pointers referenced from constructor arrays
- reset entry target from the boot header
- interrupt entry target from the boot header

### Medium Evidence

- branch targets that begin with a prologue pattern
- entries of inline jump tables that point into executable regions
- relocation targets that are known code symbols

### Weak Evidence

- aligned executable addresses reached only through heuristic scanning

Recommended rule:

- start from strong evidence
- expand through reachability
- use medium evidence to seed additional function discovery
- use weak evidence only when the binary lacks symbols and relocations

## Function-End And Return Recovery

A function end or return site may be recognized by:

- `tpop {pc}`
- `tpop {..., pc}` where the register list matches a safe return form
- `tjex lr`
- `treti` for interrupt wrappers
- a branch into a shared no-return sink

Unsafe but decodable forms shall still be represented faithfully even if they are marked suspicious.

Analysis rules:

- `tpop {r3, ..., pc}` shall decode, but it should be marked as a suspicious return form rather than a preferred canonical return
- split scratch-register returns after additional stack motion shall not be collapsed into a simple return unless proven equivalent

## Stack And Frame Recovery

The frame-analysis engine shall use the calling-convention rules from this specification.

Canonical assumptions:

- `sp` grows downward
- ordinary frame slots are 4-byte aligned
- `r4` through `r7` are callee-saved
- `lr` is preserved by non-leaf functions

### Prologue Recognition

Strong prologue patterns:

```asm
tpush {r4, r5, r6, r7, lr}
tsub  sp, #frame_size
```

```asm
tsub sp, #frame_size
```

### Epilogue Recognition

Strong epilogue patterns:

```asm
tadd sp, #frame_size
tpop {r4, r5, r6, r7, pc}
```

```asm
tadd sp, #frame_size
tpop {pc}
```

The analysis engine shall track:

- stack-pointer deltas
- saved-register slots
- outgoing-argument growth
- variadic register-save areas when present

## Condition And Flag Recovery

When reconstructing branches, the analysis module shall use the full condition table from the CPU model.

Additional recovery rules:

- `EQ` and `NE` may be treated as zero tests when sourced from result-producing operations
- `MI` may be treated as a sign-bit test when sourced from a result-producing operation
- `HS`, `LO`, `HI`, `LT`, `GT`, and `LE` shall only be elevated to unsigned/signed ordering predicates when the flag source is a compare or arithmetic operation whose carry/overflow meaning is valid
- `PL`, `GE`, and `LS` are architecturally defined but shall be marked conservative or suspicious when they appear in patterns known to be unsafe as automatic code generation

Recommended annotation:

- distinguish "architecturally valid branch meaning" from "safe code-generation pattern"

## Indirect Control Transfers

The module shall distinguish:

- indirect call veneers
- tail calls
- ordinary indirect jumps
- jump-table dispatch
- ordinary returns

### Indirect Call Veneer Recognition

Canonical veneer:

```asm
__tc32_indirect_call_r12:
    tjex r12
```

Recovery rule:

- if a direct call reaches such a veneer, classify the outer transfer as an indirect call rather than as a call to a permanently separate helper function

### Tail Calls

If the final control transfer of a function is an unconditional jump to another function entry:

- classify it as a tail call edge when the stack state matches a completed epilogue

## Thunk And Veneer Recovery

The module shall recognize at least the canonical long-call thunk shape:

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

Recovery rule:

- treat this shape as a call-range extension thunk
- present the final destination as the effective call target when the embedded word resolves to executable code

## Jump-Table Recovery

The canonical jump-table layout is:

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

Recovery rules:

- classify `.long target + 1` entries as code pointers
- subtract the stored code-state bit when constructing the effective destination address
- associate the range-check branch with the jump-table dispatch
- mark byte-sized or halfword-sized compressed jump tables as non-canonical and suspicious, because this specification forbids them as generated output

## Relocation-Aware Analysis

If relocation records are available, the analysis module shall use them to improve:

- code-pointer recognition
- data-pointer recognition
- function discovery
- jump-table recovery
- thunk destination recovery

Rules:

- `R_ARM_THM_JUMP8` indicates a short conditional branch
- `R_ARM_THM_JUMP11` indicates a short unconditional branch
- `R_ARM_THM_CALL` indicates a direct call through `tjl`
- `R_ARM_THM_JUMP24` indicates a long direct branch encoding
- `R_ARM_ABS32` indicates an absolute word reference whose interpretation depends on context

For `R_ARM_ABS32`:

- if the referenced location is used as a function pointer or callback entry, interpret the stored word as `symbol + 1`
- otherwise interpret it as a raw absolute data address unless other evidence proves it is code

## Startup And Runtime Region Recognition

The analysis module shall recognize the initial flash header at address zero.

Canonical interpretation of the first 32 bytes:

- reset-entry branch
- image version
- image magic
- encoded image-size field
- interrupt-entry branch
- manufacturer code
- image type
- binary size
- reserved field

Recovery rules:

- the target of the first branch is the reset entry
- the target of the second branch is the interrupt entry
- constructor arrays, if present, are data regions containing code pointers rather than inline code
- the early-availability code region shall be shown as executable even if it is copied or mirrored into SRAM for runtime use

## MMIO And Peripheral Annotation

The analysis environment should model the principal MMIO region as side-effectful memory.

Recommended annotations:

- interrupt-control registers
- timer registers
- GPIO registers
- analog gateway registers
- power-management retention registers

The purpose of MMIO annotation is:

- to avoid false identification of these addresses as ordinary RAM objects
- to improve decompilation readability
- to distinguish startup register programming from memory copies

## Decompilation-Oriented Type Recovery

A decompilation-oriented tool should apply the following default interpretations:

- values used as code pointers in callback arrays or indirect-call sites are function-pointer candidates
- values stored into byte or halfword memory locations are candidates for narrow integer types
- address calculations relative to `sp` are stack-variable candidates
- address calculations relative to `pc` feeding `tloadr` are literal-pool references
- branches guarding a `tmov pc, reg` dispatch are switch candidates

The tool shall not force source-language-specific constructs that are not proven by the binary.

Examples:

- do not assume one specific struct layout when multiple layouts fit the same accesses
- do not assume one specific enum type from a compare chain alone
- do not assume one specific source-language calling convention beyond the machine ABI

## Suspicious Pattern Reporting

The module should surface, but still decode, patterns that are:

- architecturally valid
- analyzable
- not safe as canonical compiler output

Examples:

- direct long conditional branches, except when they clearly serve as the mandatory repair for a short conditional edge whose resolved target is exactly `P + 4`
- direct far `tj`
- `PL`, `GE`, or `LS` branch shapes in suspicious contexts
- multi-pop returns involving `r3` and `pc`
- compressed jump-table layouts

This helps distinguish:

- "what the machine can execute"
- from "what a correct production compiler should normally emit"

## Minimum Capability Checklist

A complete TC32 binary-analysis or decompilation module shall be able to:

1. decode all documented TC32 instruction encodings
2. distinguish code from embedded data using sections and mapping symbols
3. recover direct branch and direct call targets using `pc + 4`
4. recover function starts from symbols, call targets, and startup vectors
5. recognize canonical returns, thunks, and indirect-call veneers
6. recover inline 32-bit jump tables
7. model code pointers as `address + 1` in stored form
8. interpret the boot header at flash address zero
9. model stack frames using the documented calling convention
10. annotate suspicious but decodable non-canonical patterns

## Final Recommendation

When implementing a TC32 binary-analysis module from scratch:

- begin with the exact instruction decoder
- add relocation-aware code/data separation next
- add function and CFG recovery after decoding is stable
- add jump-table, thunk, and startup recognition next
- add decompilation-oriented type and frame recovery only after the low-level model is trustworthy

That order prevents high-level recovery heuristics from masking low-level decode or control-flow errors.
