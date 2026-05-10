# 08. ELF, Relocations, Linked Images, And Flat Binaries

## Canonical ELF Identity

The canonical TC32 ELF identity is:

- machine type: `EM_TC32 = 0x8800`
- class: 32-bit ELF
- data encoding: little-endian
- object format name: `elf32-littletc32`
- canonical bare-metal target spelling: `tc32-unknown-none-elf`
- ELF header flags: `0`

## Object Section Contract

The canonical object sections are:

- `.vectors`
- `.ram_code`
- `.text`
- `.rodata`
- `.data`
- `.bss`
- `.custom_data`
- `.custom_bss`
- optional `.preinit_array`
- optional `.init_array`
- optional `.fini_array`
- `.TC32.attributes`
- optional `.TC32.exidx`

Rules:

- executable code shall be emitted into one of the executable code sections
- writable initialized storage shall be emitted into `.data` or `.custom_data`
- zero-initialized writable storage shall be emitted into `.bss` or `.custom_bss`
- constructor or early-runtime callback tables, if used, shall be emitted into `.preinit_array` or `.init_array`
- termination callback tables, if used, shall be emitted into `.fini_array`
- optional unwind metadata, if used at all, shall use the `.TC32.exidx` namespace rather than an ARM-branded namespace

## Mapping Symbols

Code sections may contain mapping symbols:

- `$t` for code
- `$d` for embedded data
- `$d.rodata` for embedded read-only data

`toolchain obligation`: disassemblers and post-link binary analyzers shall honor these symbols and must not decode data words inside a `$d` region as instructions.

## Relocation Semantics

TC32 reuses ARM relocation record names but not generic ARM execution assumptions.

| Relocation record | TC32 meaning |
| --- | --- |
| `R_ARM_THM_JUMP8` | short conditional branch |
| `R_ARM_THM_JUMP11` | short unconditional branch |
| `R_ARM_THM_CALL` | `tjl` branch-with-link |
| `R_ARM_THM_JUMP24` | long direct branch encoding |
| `R_ARM_ABS32` | absolute word reference |

## Relocation Formulas

### Short Conditional Branch

```text
target = P + 4 + sign_extend(imm8 << 1)
```

### Short Unconditional Branch

```text
target = P + 4 + sign_extend(imm11 << 1)
```

### Long Call Branch

```text
target = P + 4 + sign_extend(imm22 << 1)
```

Where:

- `P` is the address of the branch instruction
- the branch base is always `P + 4`

## Code Pointer Materialization

If a relocation resolves to a code address stored in data memory:

- store `resolved_address + 1`

If a relocation resolves to a data address stored in data memory:

- store `resolved_address`

## Linked Image Section Order

The canonical flash image order for TLSR8258-class firmware is:

1. boot and vector region
2. early-availability code region
3. ordinary executable code
4. read-only data
5. flash-resident copies of initialized writable sections

The canonical SRAM run-time order is:

1. reserved icache tag region
2. `.data`
3. `.bss`
4. `.custom_data`
5. `.custom_bss`
6. stack growing downward from the top of SRAM

## Recommended Linked Image Symbols

The following linker-defined symbols are sufficient for a complete startup runtime:

- `_start_data_`
- `_end_data_`
- `_dstored_`
- `_start_bss_`
- `_end_bss_`
- `_start_custom_data_`
- `_end_custom_data_`
- `_custom_stored_`
- `_start_custom_bss_`
- `_end_custom_bss_`
- `_ictag_start_`
- `_ictag_end_`
- `_ramcode_size_align_256_`
- `_stack_end_`
- `_bin_size_`
- `_bin_size_div_16`
- `_start_preinit_array_`
- `_end_preinit_array_`
- `_start_init_array_`
- `_end_init_array_`
- `_start_fini_array_`
- `_end_fini_array_`

## Range Extension At Link Time

If a direct call edge exceeds the range of `tjl`, the linker shall insert a call thunk.

Canonical call thunk:

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

The thunk target word is an absolute code address.

## Linked Entry Point

The linked image entry symbol is the reset-entry symbol placed at flash address `0x00000000`.

`toolchain obligation`: the linked ELF entry point and the flat binary start address shall agree.

## Flat Binary Extraction

The flat binary format for TC32 firmware is the byte-for-byte flash image beginning at flash address zero.

Rules:

- include all flash-resident bytes from the first byte of `.vectors`
- include the early-availability code region
- include ordinary `.text`
- include `.rodata`
- include the flash copies of `.data` and `.custom_data`
- do not include `NOLOAD` SRAM-only sections such as `.bss` and `.custom_bss`

The resulting binary is what a device programmer writes into flash starting at `0x00000000`.

## Dynamic Features Outside The Minimal Profile

The minimal TC32 firmware profile defined here does not require:

- dynamic linking
- position-independent executable startup
- thread-local storage
- generic unwinding tables

An implementation may add them, but they are outside the mandatory `program -> bin` contract defined by this specification.
