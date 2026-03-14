# Source_Files/CSeries/cscluts_sdl.cpp

## File Purpose
SDL-based implementation for converting Mac CLUT (Color Look-Up Table) resources to internal color_table structures. Provides global color constants used throughout the engine.

## Core Responsibilities
- Convert Mac CLUT resource binary format to color_table structures
- Define and manage global color constants (black, white, system palette)
- Handle endian-aware color data parsing from resource streams

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| RGBColor | struct | 16-bit RGB color (defined in cseries.h) |
| color_table | struct | Collection of colors with count (defined elsewhere, likely cscluts.h) |
| rgb_color | struct | Individual color entry (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rgb_black | RGBColor | global | Black (0x0000, 0x0000, 0x0000) |
| rgb_white | RGBColor | global | White (0xffff, 0xffff, 0xffff) |
| system_colors | RGBColor[NUM_SYSTEM_COLORS] | global | System palette (two grays) |

## Key Functions / Methods

### build_color_table
- **Signature:** `void build_color_table(color_table *table, LoadedResource &clut)`
- **Purpose:** Converts a Mac CLUT resource to an internal color_table
- **Inputs:**
  - `table`: pointer to color_table structure to populate
  - `clut`: LoadedResource containing CLUT binary data
- **Outputs/Return:** None (modifies *table in-place)
- **Side effects:** 
  - Allocates SDL_RWops stream for resource data
  - Modifies table->color_count and table->colors array
- **Calls:**
  - SDL_RWFromMem, SDL_RWseek, SDL_ReadBE16, SDL_RWclose
- **Notes:**
  - Format: 6-byte header skip, BE16 countΓêÆ1, then per-color: 2-byte skip + 3├ù BE16 (R, G, B)
  - Clamps count to [0, 256] range
  - Asserts SDL_RWops pointer is non-null; no other error handling

## Control Flow Notes
Resource loading phase (init-time). Converts Mac resource formats to internal engine format; no involvement in frame/render/update cycles.

## External Dependencies
- **SDL:** SDL_endian.h, SDL_RWops API (stream operations, endian-aware reads)
- **FileHandler.h:** LoadedResource class
- **cseries.h:** RGBColor definition, cscluts.h include (color_table/rgb_color definitions not visible in provided headers)
