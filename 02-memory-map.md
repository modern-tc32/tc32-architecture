# 02. Data Model, Memory Map, And Storage

## Scalar Storage Model

- The machine is byte-addressed.
- Bytes are little-endian within halfwords and words.
- A halfword is 16 bits.
- A word is 32 bits.
- Pointers are 32-bit unsigned addresses.

## Natural Alignment Rules

The canonical TC32 storage model uses:

- 1-byte alignment for byte objects
- 2-byte alignment for halfword objects
- 4-byte alignment for word objects and pointers

Generation rules:

- `safe to generate`: naturally aligned byte, halfword, and word accesses.
- `must not generate`: ordinary halfword or word memory operations to addresses that are not naturally aligned unless the implementation explicitly expands them into smaller accesses.
- `toolchain obligation`: if a front-end admits packed or misaligned objects, lower misaligned loads and stores through byte operations or helper routines.

Example:

```text
load 32-bit value from aligned address   -> one word load
load 32-bit value from unaligned address -> four byte loads plus shifts/or
```

## Aggregate Storage Model

For target-independent compiler lowering, use the following machine contract:

- aggregate alignment is the maximum alignment of its fields, capped at 4 bytes
- aggregate size is rounded up to its alignment
- arrays use element alignment and contiguous storage
- function pointers occupy one 32-bit slot and store `code_address + 1`

## Physical Address Space

The canonical TLSR8258 memory map is:

- flash program memory: `0x00000000` to `0x0007ffff`
- MMIO window: `0x00800000` to `0x0083ffff`
- SRAM: `0x00840000` to `0x0084ffff`

## SRAM Subdivision

The SRAM banks are conventionally treated as:

- `0x00840000` to `0x00841fff`: 8 KB retention SRAM bank 0
- `0x00842000` to `0x00843fff`: 8 KB retention SRAM bank 1
- `0x00844000` to `0x00847fff`: 16 KB retention SRAM bank 2
- `0x00848000` to `0x0084ffff`: 32 KB ordinary SRAM

Power-management meaning:

- deep sleep with retention preserves the first 32 KB
- deep sleep without retention does not preserve SRAM contents
- active-mode software may treat all SRAM as ordinary read/write memory

## MMIO Regions

Important MMIO regions are:

| Region | Address | Size | Purpose |
| --- | --- | --- | --- |
| interrupt controller | `0x00800640` | `0x0c` | mask, pending, enable state |
| timer block | `0x00800620` | small | timer control and status |
| system timer | `0x00800740` | `0x10` | system tick counters |
| UART0 | `0x00800090` | `0x10` | serial I/O |
| GPIO banks | `0x00800580` to `0x008005a0` | banked | pin state and direction |
| wake GPIO helper | `0x008005b5` | byte | wake and interrupt routing |
| RF and 802.15.4 block | `0x00800f00` | `0x40` | radio control |
| analog access gateway | `0x000000b8` to `0x000000ba` | indirect | analog register portal |

Generation rule:

- `safe to generate`: ordinary byte-addressed MMIO access sequences in the `0x00800000` region.
- `toolchain obligation`: do not assume MMIO addresses are cacheable or mergeable.

## Recommended Linked Storage Profile

For a 512 KB flash / 64 KB SRAM image, the canonical placement profile is:

- code and boot header start in flash at `0x00000000`
- writable data is loaded from flash but lives in SRAM
- zero-initialized storage lives only in SRAM
- stack top is placed at the top of SRAM, commonly `0x00850000`

## Dedicated Code Regions

This specification defines two executable storage classes:

- ordinary code
- early-availability code

Ordinary code:

- lives in the general executable image
- may assume normal flash execution after startup has completed

Early-availability code:

- is conventionally placed in `.ram_code`
- is reserved for routines that must remain callable during flash-control sequences, power transitions, or resume handling
- shall be placed before ordinary `.text` in the flash image profile defined later in this specification

## Data Initialization Model

The linked image distinguishes:

- load address: where initialized bytes reside in flash
- run address: where writable objects reside in SRAM after startup initialization

Sections in this model:

- `.data`: initialized writable objects, copied from flash to SRAM
- `.bss`: zero-initialized writable objects, zeroed in SRAM
- `.custom_data`: additional initialized writable region, copied from flash to SRAM
- `.custom_bss`: additional zero-initialized writable region, zeroed in SRAM

## Function And Data Pointer Rules

- Data pointers store the true byte address.
- Function pointers store `entry_address + 1`.
- Jump tables storing direct code targets shall also store `target + 1`.

Example:

```text
data object at 0x00840120  -> stored pointer 0x00840120
function at 0x00001234     -> stored pointer 0x00001235
```
