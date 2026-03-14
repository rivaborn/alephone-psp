# Source_Files/RenderMain/textures.h

## File Purpose
Header file defining bitmap texture structures and utility functions for the Aleph One game engine. Provides memory layout definitions and pixel data manipulation routines for in-game textures.

## Core Responsibilities
- Define bitmap texture data structure (`bitmap_definition`) with metadata and row addressing
- Calculate bitmap memory origin and row address tables
- Support bitmap color remapping via palette lookup tables
- Track bitmap flags (column order, transparency, patching status)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct | Core texture representation with dimensions, bytes-per-row, bit depth, flags, and row address pointers |
| Bitmap flags (enum) | enum constants | `_COLUMN_ORDER_BIT`, `_TRANSPARENT_BIT`, `_PATCHED_BIT` ΓÇö control bitmap layout and rendering behavior |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SIZEOF_bitmap_definition` | const int | file-static | Compile-time assertion that struct is 30 bytes |

## Key Functions / Methods

### calculate_bitmap_origin
- Signature: `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap)`
- Purpose: Locate the start address of bitmap pixel data, which immediately follows the `bitmap_definition` structure in memory
- Inputs: Pointer to a `bitmap_definition`
- Outputs/Return: Pointer to first pixel data byte
- Side effects: None
- Calls: Not inferable from header
- Notes: Assumes pixel data is laid out contiguously after struct; used during texture setup

### precalculate_bitmap_row_addresses
- Signature: `void precalculate_bitmap_row_addresses(struct bitmap_definition *texture)`
- Purpose: Initialize `row_addresses[]` array and validate `bytes_per_row` and `height` fields
- Inputs: Pointer to `bitmap_definition` (caller must pre-populate `bytes_per_row`, `height`, and `row_addresses[0]`)
- Outputs/Return: Void (modifies struct in-place)
- Side effects: Modifies input struct's row_addresses table
- Calls: Not inferable from header
- Notes: Precondition: caller must initialize height, bytes_per_row, and first row address before calling

### map_bytes
- Signature: `void map_bytes(byte *buffer, byte *table, long size)`
- Purpose: Apply a lookup table transformation to a byte buffer (e.g., palette remapping)
- Inputs: `buffer` (data to transform), `table` (lookup), `size` (byte count)
- Outputs/Return: Void (modifies buffer in-place)
- Side effects: In-place modification of buffer
- Calls: Not inferable from header

### remap_bitmap
- Signature: `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table)`
- Purpose: Apply color palette remapping to all pixels in a bitmap
- Inputs: `bitmap` (texture to recolor), `table` (palette lookup table)
- Outputs/Return: Void (modifies bitmap data in-place)
- Side effects: Changes pixel values in bitmap; likely calls `map_bytes` internally
- Calls: Not inferable from header
- Notes: Assumes bitmap pixel data is contiguous and accessible via row addresses

## Control Flow Notes
Bitmap setup follows: `bitmap_definition` allocation ΓåÆ `precalculate_bitmap_row_addresses()` ΓåÆ optional `remap_bitmap()` before rendering. Functions appear to be called during asset loading / initialization phases.

## External Dependencies
- `cseries.h` ΓÇö provides `pixel8`, `int16`, `byte` type definitions and cross-platform SDL setup
- Assumes pixel data layout is immediately after struct in memory (no indirection)
