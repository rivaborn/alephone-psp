# Source_Files/Misc/AlephSansMono-Bold.h

## File Purpose
Embedded TrueType font resource containing glyph data and metrics for "Aleph Sans Mono Bold" typeface. Generated from binary font file via bin2h conversion script. Enables offline font rendering without external font file dependencies.

## Core Responsibilities
- Provides complete TrueType font binary data as a static C array
- Contains glyph outlines, metrics, and character mappings
- Supplies font tables (head, hhea, hmtx, glyf, loca, cmap, etc.)
- Enables text rendering when linked into the application
- Eliminates runtime font file loading dependency

## Key Types / Data Structures
None (raw binary data array).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aleph_sans_mono_bold` | `unsigned char[]` | global | TrueType font binary data (~5543 bytes) |

## Key Functions / Methods
None. This is a data resource file containing no executable code or functions.

## Control Flow Notes
This file is a passive resource: loaded/referenced during engine initialization for font system setup. The binary array is passed to a TrueType parser/rasterizer (not visible in this file) when text rendering is needed. No control flow logic present in this file itself.

## External Dependencies
- Generated from binary font file (implied input)
- Dependency is on code that parses TTF format (defined elsewhere in engine)
- No explicit `#include` or external symbol references
- TrueType format structures implied but not defined here

---

**Notes:**
- File timestamp: 2008-05-03 (pre-generated, not source code)
- Embedded font enables shipping without separate font assets
- TTF table directory visible in byte offsets 12ΓÇô68 (FFTM, OS/2, cmap, etc.)
- Binary contains both glyph glyphs and licensing text (visible near end in UTF-16 encoding)
