# Source_Files/Misc/key_definitions.h

## File Purpose
Defines static keyboard-to-action mappings for the game engine across three input layout presets (standard QWERTY, left-handed, PowerBook). Provides hardware key codes (SDL or native) paired with game action flags, used during input initialization and frame processing.

## Core Responsibilities
- Define enum constants for special flag types (`_double_flag`, `_latched_flag`)
- Declare data structures for key mappings, blacklisted key combinations, and special flag metadata
- Provide three static key definition arrays for different keyboard layouts
- Maintain a master array of pointers to all layout configurations
- Export the current active key definition array for vbl.c and vbl_macintosh.c to consume

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `blacklist_data` | struct | Stores pairs of key offsets and masks to identify and filter blacklisted key combinations |
| `special_flag_data` | struct | Metadata for special flags including type, flag values, and persistence duration |
| `key_definition` | struct | Maps a hardware key code (SDLKey or native offset) to a game action flag; SDL vs. native branch |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `standard_key_definitions` | `struct key_definition[]` | static | Default QWERTY keypad/modifier layout (23 entries) |
| `left_handed_key_definitions` | `struct key_definition[]` | static | Left-handed arrow-key layout (23 entries) |
| `powerbook_key_definitions` | `struct key_definition[]` | static | PowerBook optimized layout (23 entries) |
| `all_key_definitions` | `struct key_definition*[]` | static | Pointer array to the three layouts; size determined by `NUMBER_OF_KEY_SETUPS` |
| `current_key_definitions` | `struct key_definition[]` | extern | Active layout; populated at runtime by vbl.c or vbl_macintosh.c |
| `NUMBER_OF_STANDARD_KEY_DEFINITIONS` | macro | static | Computed array size for standard layout |
| `NUMBER_OF_LEFT_HANDED_KEY_DEFINITIONS` | macro | static | Computed array size for left-handed layout |
| `NUMBER_OF_POWERBOOK_KEY_DEFINITIONS` | macro | static | Computed array size for PowerBook layout |

## Key Functions / Methods
None. This is a pure data definition header.

## Control Flow Notes
This file is loaded during engine initialization. The three static arrays define default key mappings; at runtime, one is selected and copied/referenced as `current_key_definitions` by input handling code (vbl.c). The layout selection likely occurs in a setup/configuration routine before the main game loop begins.

## External Dependencies
- **Conditional includes**: SDL library (SDL.h implied) when `SDL` is defined; otherwise native platform key codes (likely macOS)
- **External symbols used** (defined elsewhere): action flag constants (`_moving_forward`, `_looking_left`, `_left_trigger_state`, `_toggle_map`, `_microphone_button`, `_cycle_weapons_forward`, etc.)
- **Consumers**: vbl.c, vbl_macintosh.c reference `current_key_definitions`

---

**Notes**:  
- The `#ifdef SDL` / `#else` branching allows dual compilation for SDL and native macOS key codes.
- Comment at line ~26 notes this header should only be included by one file (vbl.c), suggesting tight coupling.
- All three layouts map the same 23 actions; only the physical key assignments differ.
