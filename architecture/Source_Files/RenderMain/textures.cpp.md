# Source_Files/RenderMain/textures.cpp

## File Purpose
Bitmap and texture memory management utilities for the Aleph One game engine. Handles bitmap origin calculation, row address precalculation for both linear and RLE-compressed formats, and palette/color remapping operations.

## Core Responsibilities
- Calculate the memory origin (starting address) of bitmap pixel data within a bitmap_definition structure
- Precalculate row address lookup tables for efficient row access in both row-order and column-order layouts
- Support both linear (standard) and RLE-compressed (transparent shape) bitmap formats
- Handle Marathon 1 and Marathon 2 RLE format variants with different endianness conventions
- Apply color/palette remapping to entire bitmaps via byte-by-byte lookup table substitution

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct (defined in textures.h) | Metadata for a bitmap: width, height, bytes per row, flags, and row address table |
| `pixel8` | typedef | 8-bit pixel value (palette index) |

## Global / File-Static State
None.

## Key Functions / Methods

### calculate_bitmap_origin
- **Signature:** `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap)`
- **Purpose:** Compute the memory address where bitmap pixel data begins.
- **Inputs:** Pointer to `bitmap_definition`
- **Outputs/Return:** Pointer to first pixel byte (`pixel8 *`)
- **Side effects:** None (read-only calculation)
- **Calls:** None
- **Notes:** The bitmap definition struct is immediately followed in memory by a row_addresses table (size varies by column/row order flag), then the pixel data. This function skips past the table to find pixel data start.

### precalculate_bitmap_row_addresses
- **Signature:** `void precalculate_bitmap_row_addresses(struct bitmap_definition *bitmap)`
- **Purpose:** Populate the bitmap's row_addresses lookup table to map each row to its starting byte address.
- **Inputs:** Bitmap with `bytes_per_row`, `height`, and `row_addresses[0]` already initialized
- **Outputs/Return:** None (modifies `bitmap->row_addresses` in-place)
- **Side effects:** Writes row address pointers into the bitmap structure
- **Calls:** None
- **Notes:** Handles two formats: (1) linear with constant `bytes_per_row`, (2) RLE-compressed where `bytes_per_row == NONE`. RLE support differs between Marathon 1 (endian-unsafe) and Marathon 2 (big-endian first/last markers). Marathon 2 variant reads 16-bit big-endian values for row extent boundaries.

### remap_bitmap
- **Signature:** `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table)`
- **Purpose:** Apply a color/palette remap to every pixel in the bitmap using a lookup table.
- **Inputs:** Bitmap and remap table (indexed by old pixel value; returns new value)
- **Outputs/Return:** None (modifies bitmap pixels in-place)
- **Side effects:** Alters all pixel data in the bitmap
- **Calls:** `map_bytes()`
- **Notes:** Works with both linear and RLE formats. For RLE, navigates via row_addresses and applies remap to the pixel data portion of each run.

### map_bytes
- **Signature:** `void map_bytes(register byte *buffer, register byte *table, register long size)`
- **Purpose:** Perform fast byte-by-byte table lookup remapping on a buffer.
- **Inputs:** Buffer to remap, lookup table, buffer size
- **Outputs/Return:** None (modifies buffer in-place)
- **Side effects:** Overwrites buffer bytes
- **Calls:** None
- **Notes:** Hot utility function; uses `register` keyword for optimization. Each byte `buffer[i]` is replaced with `table[buffer[i]]`.

## Control Flow Notes
This file provides pure utility functions for texture initialization and runtime remapping. Not part of a frame/render loop; called during asset loading and palette changes. `map_bytes()` is performance-sensitive and called per-pixel.

## External Dependencies
- **Notable includes:** `cseries.h` (base types, macros), `textures.h` (bitmap_definition)
- **Conditional compilation:** `#ifdef MARATHON1` / `#ifdef MARATHON2` for format-specific RLE handling
- **External symbols used:** `bitmap_definition`, `pixel8`, `byte`, `NONE` macro (likely -1), `_COLUMN_ORDER_BIT` flag
