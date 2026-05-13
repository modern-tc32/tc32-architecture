# 13. Compiler Implementation Blueprint

## Purpose

This chapter is the implementation plan for building a new TC32 compiler toolchain from scratch.

It assumes no reuse of an existing TC32 compiler. It does assume that the target behavior defined in the rest of this specification is the normative contract.

## Sufficiency Assessment

With the addition of the complete encoding reference and the linked-image/startup contract, the documentation in this directory is sufficient for the TC32-specific part of a full `program -> bin` toolchain.

What it now covers fully:

- target data model
- machine instruction encodings
- legal and illegal code-generation patterns
- calling convention
- relocation and object semantics
- linked-image layout
- startup and binary image contract

What remains implementation-defined:

- source language grammar
- source language type checker
- source language standard library
- source language macro or package system
- debug-info format beyond basic object generation

That is the correct boundary for a language-agnostic TC32 target specification.

## Recommended Overall Architecture

Build the toolchain as six subsystems:

1. front-end
2. canonical machine-independent IR
3. TC32 code generator
4. assembler and object writer
5. linker and image builder
6. startup runtime and support library

Each subsystem should be independently testable.

## Subsystem 1: Front-End

The front-end may target any source language, but it should lower into a common machine-independent IR with the following properties:

- explicit integer widths
- explicit pointer arithmetic
- explicit load and store alignment
- explicit calling convention boundaries
- explicit control-flow graph
- explicit aggregate copy operations
- explicit helper-call nodes for operations the base ISA cannot express directly

The front-end must not assume:

- native atomic operations
- native floating-point instructions
- unrestricted unaligned memory access
- native compressed jump tables

## Subsystem 2: Canonical IR

The canonical IR should be simple enough that a backend can reason about every machine-sensitive case.

Required IR operations:

- integer arithmetic
- bitwise logic
- comparisons
- shifts
- sign and zero extension
- truncation
- loads and stores with alignment metadata
- calls and returns
- conditional and unconditional branches
- switch
- block copy and fill
- constant address materialization

Recommended IR prohibitions before instruction selection:

- no implicit atomics
- no unresolved aggregate passing convention
- no target-ambiguous switch representation
- no opaque inline assembly that bypasses target validation

## Subsystem 3: TC32 Code Generator

The code generator should be split into the following phases.

### Phase 3.1: Legalization

Convert IR operations into forms the target can actually express.

Required actions:

- replace unsupported atomics with hard errors
- lower floating-point operations to helper calls if the language admits them
- lower division and remainder to helper calls if not handled inline
- scalarize misaligned halfword and word accesses
- scalarize packed sub-byte extraction patterns into byte loads when needed

### Phase 3.2: Calling-Convention Assignment

Assign arguments, returns, and call-preserved registers according to the ABI chapter.

Required outputs:

- register assignments for the first four argument words
- stack slots for overflow arguments
- hidden result pointer for large returns
- vararg register-save area when needed

### Phase 3.3: Instruction Selection

Select concrete TC32 instruction forms.

Required selection rules:

- prefer low-register forms
- use literal loads for in-range constants and addresses
- use helper calls or synthesized sequences for unsupported large immediates
- use only safe branch forms
- use word, halfword, and byte memory forms according to alignment

### Phase 3.4: TC32-Specific Canonicalization

Run TC32-only repair passes after initial selection.

Required passes:

- immediate arithmetic expansion
  - rewrite unsafe `tadd #imm` and `tsub #imm` pointer arithmetic into register-form arithmetic
- signed branch fixup
  - rewrite unsafe `GE`, `PL`, and `LS` condition shapes into safe CFG forms
- packed byte load/store scalarization
  - prefer byte operations for packed byte extraction
- IR/system-instruction fixup
  - reject unsupported system and atomic forms

### Phase 3.5: Register Allocation

Use low-register pressure as the primary cost model.

Required heuristics:

- keep values in `r0` through `r7` whenever possible
- avoid introducing high-register forms unless explicitly supported
- spill to word-aligned stack slots

### Phase 3.6: Frame Lowering

Generate:

- prologue
- local frame allocation
- outgoing-argument stack growth
- epilogue

Must preserve:

- 4-byte stack alignment
- safe return patterns
- return-value liveness through epilogue construction

### Phase 3.7: Control-Flow Repair

Perform:

- short-branch range checking
- exact-`P + 4` short-conditional handling
- conditional-branch rewriting
- far-edge veneer insertion
- jump-table finalization

This phase is mandatory because branch safety depends on final layout.

### Phase 3.8: Hazard Repair

Run a final machine pass that inserts:

- two `nop` after an immediately consumed load
- one `nop` after a load with one independent gap
- two `nop` after an immediately consumed `tpop`

Do this after final scheduling decisions.

## Subsystem 4: Assembler And Object Writer

This subsystem should be implemented even if the compiler primarily emits machine code directly, because:

- startup code may be hand-written
- support libraries may include assembly
- object emission needs relocation and section machinery anyway

Required components:

- parser for TC32 mnemonics and directives
- encoder using the complete instruction reference
- range diagnostics for direct source spellings
- symbol table
- section table
- mapping symbol emission
- relocation emission
- ELF writer with `EM_TC32`

## Subsystem 5: Linker And Image Builder

The linker stage must:

- place sections into flash and SRAM
- apply relocations using TC32 formulas
- build long-call thunks
- publish startup symbols
- emit a correct ELF entry point

The flat-binary stage must:

- extract only flash-resident bytes
- preserve the boot header at the start
- omit `NOLOAD` SRAM sections

## Subsystem 6: Startup Runtime And Support Library

This subsystem provides the minimum execution environment for the generated program.

Required contents:

- reset entry
- interrupt wrapper
- memory copy and zero loops
- resume-versus-cold-boot decision logic
- helper routines required by the code generator

Minimum helper set:

- block copy
- block move
- block fill
- integer division and remainder
- wide integer helpers
- floating-point helpers if the language admits floating point

## Recommended Build Order

The least risky implementation order is:

1. define IR and ABI boundary
2. implement encoder and decoder from the complete instruction reference
3. implement assembler diagnostics and relocations
4. implement object writer
5. implement startup runtime and linker script generator
6. implement ELF linker and flat-binary writer
7. implement basic instruction selection for straight-line code
8. implement register allocation and frame lowering
9. implement branch rewriting and veneers
10. implement jump tables
11. implement hazard repair
12. integrate helper library lowering

## Milestone Plan

### Milestone 1: Machine Core

Deliverables:

- instruction encoder
- instruction decoder
- assembler
- disassembler

Acceptance conditions:

- all fixed encodings match the specification
- all branch formulas and ranges are correct

### Milestone 2: Object And Link

Deliverables:

- relocatable object writer
- linker
- flat-binary emitter

Acceptance conditions:

- linked image has correct section order
- long-call thunk insertion works
- flat binary is bootable in the canonical image format

### Milestone 3: Straight-Line Compiler

Deliverables:

- scalar arithmetic
- loads and stores
- calls and returns
- startup runtime

Acceptance conditions:

- simple functions compile and execute correctly
- function pointers use `address + 1`

### Milestone 4: Full Control Flow

Deliverables:

- branches
- veneers
- jump tables
- condition fixups

Acceptance conditions:

- all safe branch patterns are generated
- no forbidden direct long conditional branches appear
- the resolved 16-bit short conditional `P + 4` case is accepted as valid generated code
- if any pass still rewrites that case, it preserves the original condition code exactly; in particular, if fixup machinery operates on a pointer already positioned at the branch instruction, it shall recover the condition from the first halfword at that pointer and shall not re-apply the fragment-local offset
- no pass shall synthesize a direct long conditional branch merely because a short conditional edge resolved to `P + 4`

### Milestone 5: Production Safety

Deliverables:

- hazard repair
- packed-byte scalarization
- runtime helpers
- misalignment handling

Acceptance conditions:

- no forbidden patterns remain in final machine code
- helper calls obey the calling convention

## Architecture Rules For A “Unique” Compiler

If the goal is a new independent compiler rather than a clone of an existing design, the best approach is:

- define one small target-neutral IR
- keep all TC32-specific rules in a dedicated backend package
- keep assembler and linker independent from the frontend
- share one object format and one startup runtime across all frontends

That separation makes it possible to support both a C-like frontend and a Rust-like frontend without duplicating any TC32-specific logic.

## Final Recommendation

Do not begin with parsing a high-level language.

Begin with:

1. encoder/decoder
2. object writer
3. linker
4. startup runtime
5. machine-code code generator
6. only then a full language frontend

That order gives a working `IR -> bin` path early and prevents frontend work from outrunning unresolved target semantics.
