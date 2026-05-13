# 10. Full Compiler Pipeline Checklist

## Goal

A complete TC32 implementation must be able to lower an abstract program into:

1. machine instructions
2. a relocatable object file
3. a linked ELF image
4. a flat binary ready to write into flash

## Stage 1: Target Data Model

Implement:

- 32-bit little-endian integer and pointer model
- byte-addressed memory
- 4-byte natural word alignment
- code-pointer representation with stored bit 0 set

## Stage 2: Instruction Selection

Implement explicit lowering for:

- register and immediate moves
- loads and stores
- arithmetic and compare
- shifts and rotates
- push and pop
- short branches
- `tjl`
- `tjex`
- literal loads
- `tmcsr`, `tmssr`, `tmrss`, `treti`, `nop`

Do not rely on generic ARM assumptions without restating the TC32 rule explicitly in the implementation.

If the source language admits operations outside the base integer ISA, also implement helper lowering for:

- floating-point arithmetic
- integer division and remainder
- wide integer arithmetic
- block copy and block fill

## Stage 3: Register Allocation

Implement:

- low-register-biased allocation
- explicit handling of the limited high-register instruction surface
- callee-saved preservation for `r4` through `r7`

## Stage 4: Frame Lowering

Implement:

- word-aligned stack frames
- per-call outgoing stack growth
- safe epilogues
- result preservation across epilogue formation
- variadic register-save area when required by the front-end

## Stage 5: Control-Flow Lowering

Implement:

- short conditional branches as the primary conditional form
- short unconditional branches as the primary local branch form
- long calls through `tjl`
- far-edge rewriting through veneers or helper blocks
- jump-table lowering through inline 32-bit target tables

Do not generate:

- direct long conditional branches as the default conditional-lowering strategy
- direct far `tj` as the default far-edge strategy
- `tjpl` for signed nonnegative tests after arbitrary helpers
- `tjls` as the main switch bound check

Mandatory repair:

- detect the resolved short-conditional case whose target is exactly `P + 4`
- emit the legal 16-bit zero-displacement encoding for that case when the short conditional edge is otherwise correct
- do not reject that case in the assembler, MC relaxer, or late fixup code
- optional canonicalization: rewrite local layout or CFG if the compiler has an independent reason to do so
- forbidden repair: synthesize a direct long conditional branch for that edge

## Stage 6: Hazard Repair

Implement a post-selection hazard pass that:

- inserts two `nop` after a load when the loaded register is used immediately
- inserts one `nop` when one independent instruction separates the load and the use
- inserts two `nop` after `tpop` when the restored register is used immediately

## Stage 7: Assembly Emission

Implement emission for:

- TC32 mnemonics
- `.code 16`
- `.tc32_func`
- canonical section names
- `.align`, `.word`, `.short`
- mapping symbols

The assembler must reject out-of-range direct spellings instead of silently rewriting them.
The resolved short-conditional `P + 4` case is not an exception to validity: it is a legal short-branch encoding with displacement zero and shall be accepted as such.

## Stage 8: Object Writer

Implement:

- `EM_TC32`
- `elf32-littletc32`
- TC32 section naming
- TC32 mapping symbols
- reused ARM relocation record names with TC32 semantics

## Stage 9: Linker

Implement:

- section placement into flash and SRAM
- relocation application with `pc + 4` bias
- call range extension through the canonical thunk sequence
- linked entry point at reset entry
- linker-defined startup symbols for copy and zero loops

## Stage 10: Flat Binary Writer

Implement:

- extraction of the contiguous flash image
- exclusion of `NOLOAD` SRAM sections
- preservation of the boot header at the start of the binary

## Stage 11: Startup Runtime

Implement:

- reset entry
- stack initialization
- writable-data copy
- zero-initialized-data clearing
- optional retention-resume path
- call into the ordinary program entry routine
- non-returning fallback loop

## Stage 12: Optional Secondary Tools

A production-grade TC32 toolchain should also implement:

- disassembly with mapping-symbol awareness
- backward-branch target reconstruction
- correct function-pointer display with code-state bit handling

## Final “Must Not Generate” List

- short-immediate pointer increment and decrement
- direct long conditional branches
- default far direct `tj`
- `tpop {r3, ..., pc}`
- split scratch-register returns after additional stack movement
- byte or halfword compressed jump tables
- unaligned halfword or word accesses without expansion
- general atomic operations

## Minimum End-To-End Bring-Up Order

If implementing from scratch, the shortest practical order is:

1. encoder and decoder for core 16-bit instructions
2. assembler and object writer
3. short-branch and `tjl` relocations
4. prologue and epilogue lowering
5. startup runtime and linker script
6. flat binary writer
7. hazard pass
8. veneers and long-call thunks
9. jump tables
10. helper-library integration
