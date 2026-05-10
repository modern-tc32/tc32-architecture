# 09. Startup, Reset, Interrupt Entry, And Runtime Initialization

## Bootable Image Header

A canonical bootable TC32 flash image for TLSR8258-class hardware starts at `0x00000000` and reserves the first 32 bytes for the boot and interrupt header.

Recommended layout:

| Offset | Size | Purpose |
| --- | --- | --- |
| `0x00` | 4 | branch to reset entry |
| `0x04` | 4 | image version word |
| `0x08` | 4 | image magic |
| `0x0c` | 4 | encoded image size field |
| `0x10` | 4 | branch to interrupt entry |
| `0x12` | 2 | manufacturer code |
| `0x14` | 2 | image type |
| `0x18` | 4 | full binary size in bytes |
| `0x1c` | 4 | reserved or implementation-defined |

One commonly used magic value is:

```text
0x544c4e4b
```

The encoded image size field at offset `0x0c` commonly uses:

```text
0x00880000 + ceil(binary_size_bytes / 16)
```

The full-binary-size field at offset `0x18` commonly uses:

```text
binary_size_bytes
```

Canonical header skeleton:

```asm
__start:
    tj   __reset
    .word image_version
    .word 0x544c4e4b
    .word 0x00880000 + ceil(binary_size_bytes / 16)
    tj   __irq
    .short manufacturer_code
    .short image_type
    .word binary_size_bytes
```

## Reset Entry Contract

The reset entry routine shall:

1. initialize `sp` from `_stack_end_`
2. branch or call into the boot-initialization routine
3. never return to an undefined location

Canonical shape:

```asm
ldr r0, =_stack_end_
mov sp, r0
tjl __tc32_boot_init
```

If the implementation uses a direct branch instead of a call, it must preserve equivalent behavior.

## Interrupt Entry Contract

A minimal interrupt entry wrapper shall:

1. preserve the return linkage
2. call the ordinary interrupt handler
3. return through a safe return form

Canonical wrapper:

```asm
tpush {lr}
tjl   irq_handler
tpop  {pc}
```

If interrupt state requires special return semantics, use `treti` in the final interrupt-return sequence instead of a normal function return.

## Boot Initialization Responsibilities

The boot-initialization routine is a `toolchain obligation` for a complete firmware image.

It shall perform the following actions in a hardware-safe order:

1. initialize any dedicated instruction-cache tag region used by early-availability code
2. bring flash access into a usable state
3. decide whether the current entry is a cold boot or a retention/deep-sleep resume
4. on cold boot:
   - copy `.data` from flash load address to SRAM run address
   - zero `.bss`
   - copy `.custom_data`
   - zero `.custom_bss`
   - execute `.preinit_array` callbacks in ascending address order
   - execute `.init_array` callbacks in ascending address order
5. on retention resume:
   - preserve retained SRAM contents that must survive sleep
   - avoid reinitializing regions that are intentionally retained
6. transfer control to the language-independent program entry routine

## Program Entry Contract

After memory initialization is complete, startup shall call the ordinary program entry routine as a normal TC32 function.

Minimal contract:

- arguments: none required
- result: ignored
- if the entry routine returns, startup shall enter a non-returning halt or spin loop

Canonical pattern:

```asm
tjl main_entry
.Lhang:
tj .Lhang
```

If the runtime supports termination callbacks, it may execute `.fini_array` only on an orderly program exit path.
For the minimal non-returning firmware profile, `.fini_array` support is optional.

## SRAM Initialization Rules

### Initialized Writable Data

For each initialized writable section:

- source bytes are stored in flash
- destination bytes live in SRAM
- startup copies the bytes word by word or byte by byte

### Zero-Initialized Writable Data

For each zero-initialized writable section:

- startup writes zero to the full section range in SRAM before program entry

## Initialization Callback Arrays

If the implementation supports startup callback arrays, each entry is one 32-bit code pointer word using the normal stored code-pointer representation:

```text
stored callback value = function_address + 1
```

Execution rules:

- `.preinit_array` runs after writable-memory initialization and before `.init_array`
- `.init_array` runs before the ordinary program entry routine
- callbacks run in ascending address order
- a null entry may be skipped
- callback calls use the ordinary TC32 calling convention

Minimal callback walker shape:

```asm
; r0 = current entry pointer
; r1 = end pointer
tpush {r4, r5, lr}
tmov  r4, r0
tmov  r5, r1
.Lwalk:
tcmp  r4, r5
tjhs  .Ldone
tloadr r2, [r4, #0]
nop
nop
tcmp  r2, #0
tjeq  .Lnext
tmov  r12, r2
tjl   __tc32_indirect_call_r12
.Lnext:
tmov  r3, #4
tadd  r4, r3
tj    .Lwalk
.Ldone:
tpop  {r4, r5, pc}
```

### Optional Stack Fill

An implementation may fill unused stack space with a known pattern during cold boot for overflow diagnostics.

This is optional and not part of the callable ABI.

## Early-Availability Code Region

The early-availability code region is used for routines that must remain available while ordinary flash access is being configured or while power-management transitions are in progress.

Rules:

- place these routines in `.ram_code`
- align the reserved size to 256 bytes for icache-tag bookkeeping
- provide `_ramcode_size_align_256_`
- reserve an icache tag region bounded by `_ictag_start_` and `_ictag_end_`

## Recommended Canonical Layout For 64 KB SRAM

The following profile is a practical and internally consistent layout:

- flash base: `0x00000000`
- SRAM base: `0x00840000`
- icache tag region starts at `0x00840000`
- ordinary writable data begins after the aligned early-code reservation, commonly from `0x00840900`
- stack top: `0x00850000`

This profile is not the only possible one, but a fresh implementation should match it unless there is a strong reason not to.

## Image Size Fields

The boot header commonly carries:

- total binary size in bytes
- binary size divided by 16 and rounded up

Those values are used by programming and boot logic that reasons about flash image extent.

## Cold Boot Versus Resume

Startup must distinguish at least two cases:

- cold boot
- retention or deep-sleep resume

Cold boot rules:

- fully initialize writable memory
- initialize startup-owned state

Resume rules:

- preserve retained SRAM contents
- avoid destroying wakeup or power-management context that the resumed program expects

## Final Startup Rule

A TC32 binary is incomplete unless it defines:

- a flash-resident reset entry at address zero
- a linked stack-top symbol
- a boot routine that initializes writable memory
- a non-returning fallback path if the program entry routine returns
