# Source_Files/CSeries/cscluts.h

## File Purpose
Defines color lookup table (CLUT) data structures and provides functions for building and managing color tables and system colors in the Aleph One game engine. Serves as the interface between the engine's color management system and platform-specific color implementations.

## Core Responsibilities
- Define color value structures (individual RGB colors and lookup tables)
- Declare platform-specific color table construction (`build_macintosh_color_table`)
- Declare portable color table construction (`build_color_table`)
- Enumerate and export system color constants
- Provide predefined color constants (black, white)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | Single RGB color with 16-bit per-channel (red, green, blue) |
| `color_table` | struct | Lookup table containing a count (0ΓÇô256) and array of rgb_color entries |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rgb_black` | RGBColor | global | Predefined black color constant |
| `rgb_white` | RGBColor | global | Predefined white color constant |
| `system_colors` | RGBColor[] | global | Array of NUM_SYSTEM_COLORS system palette colors (e.g., gray15Percent, windowHighlight) |

## Key Functions / Methods

### build_macintosh_color_table
- Signature: `CTabHandle build_macintosh_color_table(color_table *table)`
- Purpose: Convert a portable `color_table` into a native Macintosh color table handle
- Inputs: Pointer to a `color_table` struct
- Outputs/Return: `CTabHandle` (Mac-native color table handle)
- Side effects: Allocates Mac OS resources
- Notes: Mac-only (`#ifdef mac`); used during initialization or resource loading

### build_color_table
- Signature: `void build_color_table(color_table *table, LoadedResource &clut)`
- Purpose: Populate a `color_table` struct from a loaded resource (likely a CLUT resource)
- Inputs: Pointer to destination `color_table`; reference to a `LoadedResource` containing CLUT data
- Outputs/Return: Void; modifies `table` in place
- Side effects: Parses resource data, populates `table`
- Notes: Platform-independent; used during resource loading (init phase)

## Control Flow Notes
This is a data structure and interface definition. Functions are called during initialization when resources are loaded to populate color tables for rendering operations.

## External Dependencies
- **Includes:** `cstypes.h` (for `uint16`, `int16` types)
- **Forward declare:** `LoadedResource` class
- **Defined elsewhere:** `RGBColor` type (platform-specific, likely QuickDraw on Mac or SDL color struct on other platforms); implementations of `build_macintosh_color_table` and `build_color_table`
