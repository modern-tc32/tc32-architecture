# TC32 Machine And Toolchain Specification

This directory is a language-agnostic specification for producing correct TC32 binaries for TLSR8258-class hardware.

It is written for implementers of:

- compilers
- assemblers
- object writers
- linkers
- flat-binary emitters
- disassemblers
- firmware startup runtimes

The document set describes the full path from an abstract program to a bootable binary image.

## Normative Vocabulary

Every rule in this specification uses one of the following meanings:

- `architecturally defined`: the encoding or behavior exists in the TC32 machine model.
- `safe to generate`: an automatic code generator may emit this form for production code on TLSR8258-class hardware.
- `must not generate`: the encoding may exist, but a compiler, assembler relaxer, or linker must not choose it automatically for production code.
- `toolchain obligation`: a required action by the compiler, assembler, linker, startup runtime, or binary writer.

The distinction between `architecturally defined` and `safe to generate` is central. TC32 is ARM/Thumb-like, but code generation must follow TC32-specific rules explicitly stated here. No behavior is implied solely by ARM naming.

## Scope

This specification covers:

- CPU execution model
- register file and flags
- memory map and storage model
- machine instruction forms and decode rules
- safe and unsafe lowering patterns
- calling convention and stack discipline
- branch ranges, veneers, jump tables, and hazard padding
- assembly syntax and object emission
- ELF identity, relocation semantics, linked-image layout, and flat-binary extraction
- startup, reset entry, interrupt entry, and runtime initialization
- end-to-end compiler pipeline requirements

This specification does not define any source language. It defines the target contract that any language front-end must obey when producing TC32 code.

## Document Set

- [01-cpu-and-programming-model.md](01-cpu-and-programming-model.md)
- [02-memory-map.md](02-memory-map.md)
- [03-instruction-set.md](03-instruction-set.md)
- [04-assembly-syntax.md](04-assembly-syntax.md)
- [05-abi-and-codegen.md](05-abi-and-codegen.md)
- [06-branches-jump-tables-and-islands.md](06-branches-jump-tables-and-islands.md)
- [07-pipeline-hazards.md](07-pipeline-hazards.md)
- [08-elf-linking-and-relocations.md](08-elf-linking-and-relocations.md)
- [09-startup-reset-interrupt-and-runtime-initialization.md](09-startup-reset-interrupt-and-runtime-initialization.md)
- [10-compiler-backend-checklist.md](10-compiler-backend-checklist.md)
- [11-worked-examples-and-conformance-scenarios.md](11-worked-examples-and-conformance-scenarios.md)
- [12-complete-instruction-encoding-reference.md](12-complete-instruction-encoding-reference.md)
- [13-compiler-implementation-blueprint.md](13-compiler-implementation-blueprint.md)
- [14-full-source-to-binary-compiler-plan.md](14-full-source-to-binary-compiler-plan.md)

## Reading Order

1. CPU execution model.
2. Data model, memory map, and storage rules.
3. Machine instructions, encodings, and semantics.
4. Calling convention and control-flow lowering rules.
5. Object, relocation, image, and startup rules.
6. Complete machine-code encoding reference.
7. Worked examples, compiler blueprint, and end-to-end checklist.
8. Full source-to-binary compiler plan, legalization matrix, and runtime-support contract.

## Design Rule

If a code generator can choose between a compact encoding and a more conservative encoding, it shall prefer the conservative encoding whenever the compact encoding is not explicitly marked `safe to generate`.
