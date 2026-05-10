# 04. Assembly Language, Directives, And Object Emission

## Assembly Surface

The canonical TC32 assembly surface uses vendor-style `t`-prefixed mnemonics.

Examples:

```asm
tj    label
tjl   function
tjex  lr
tloadr r0, [r1, #4]
tstorer r2, [sp, #8]
```

## Required Mnemonic Set

A TC32 assembler shall accept the following base families:

- control flow:
  - `tj`
  - `tjl`
  - `tjex`
  - `tjeq`, `tjne`, `tjhs`, `tjlo`, `tjmi`, `tjpl`, `tjvs`, `tjvc`, `tjhi`, `tjls`, `tjge`, `tjlt`, `tjgt`, `tjle`
- arithmetic and compare:
  - `tadd`, `taddc`, `tsub`, `tsubc`, `tcmp`, `tcmpn`, `tmul`
- logical:
  - `tand`, `tor`, `txor`, `tbclr`
- moves and shifts:
  - `tmov`, `tshftl`, `tshftr`, `tasr`, `trotr`
- memory:
  - `tloadr`, `tloadrb`, `tloadrh`, `tloadrsb`, `tloadrsh`
  - `tstorer`, `tstorerb`, `tstorerh`
- stack and system:
  - `tpush`, `tpop`, `tmcsr`, `tmssr`, `tmrss`, `treti`, `nop`

## Alias Rules

The canonical printed spellings are TC32 spellings even if alternative source aliases are accepted.

Required alias behavior:

- print `tor` instead of plain `orr`
- print `tmrss` even if the source spelling was `tmrcs`
- preserve `tshftl` versus `tshftr` distinctly

## Literal-Load Source Syntax

The assembler shall accept both explicit and vendor-style literal-load syntax.

Accepted forms:

```asm
tloadr   r0, table
tloadr   r1, table + 4
tloadrb  r2, table+8
tloadrh  r3, table+12
tloadrsb r4, table+16
tloadrsh r5, table+20
tloadr   r6, =table+24
```

Rules:

- label expressions attached to suffixed load mnemonics are lowered through the literal-load encoding when appropriate
- the assembler shall diagnose out-of-range literal targets rather than inventing a different instruction sequence

## Function And Mode Directives

The following directives are part of the canonical TC32 assembly contract:

- `.code 16`
- `.tc32_func`
- `.align`
- `.short`
- `.word`
- `.section`

Rules:

- `.code 16` selects TC32 halfword-mode assembly.
- `.tc32_func` marks a symbol as a function entry in TC32 code state.
- `.align` is the canonical alignment directive for emitted assembly.

## Section Classes

The canonical section classes for TC32 firmware are:

- `.vectors`
- `.ram_code`
- `.text`
- `.rodata`
- `.data`
- `.bss`
- `.custom_data`
- `.custom_bss`
- `.TC32.attributes`
- optional `.TC32.exidx`

`toolchain obligation`: an object writer shall preserve the section class of each fragment so that the linker can apply the final image layout rules later in this specification.

## External Symbol Address Rules

When a code symbol is referenced in data:

- store `symbol + 1`

When a data symbol is referenced in data:

- store `symbol`

Example:

```asm
.word handler + 1
.word global_data
```

## Range Diagnostics

The assembler shall reject source branches and literal loads that exceed the encodable range of the written instruction.

Required diagnostics classes:

- out-of-range short conditional branch
- out-of-range short unconditional branch
- out-of-range direct call branch
- out-of-range literal load
- unsupported system instruction

`toolchain obligation`: the assembler shall not silently invent a veneer, long form, or helper sequence for an instruction that the source explicitly wrote in a shorter form.

## Relocations Emitted By Assembly

The object writer uses the following relocation meanings:

| Relocation record | Meaning |
| --- | --- |
| `R_ARM_THM_JUMP8` | short conditional branch |
| `R_ARM_THM_JUMP11` | short unconditional branch |
| `R_ARM_THM_CALL` | `tjl` direct call branch |
| `R_ARM_THM_JUMP24` | long direct branch encoding |
| `R_ARM_ABS32` | absolute 32-bit word reference |

The record names remain in the reused ARM namespace, but the relocation semantics are TC32 semantics.

## Mapping Symbols

Object writers and assemblers shall support mapping symbols in code sections:

- `$t` marks code
- `$d` marks data
- `$d.rodata` marks embedded read-only data

These symbols are part of the object contract used by disassemblers and post-link inspection tools.
