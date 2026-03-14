# Source_Files/RenderOther/fades.cpp

## File Purpose
Implements the game's screen fade and color-effect system. Manages timed fade transitions (cinematic fades, damage flashes, color tints) and applies various color-manipulation effects to the game's palette. Supports XML configuration and OpenGL rendering backends.

## Core Responsibilities
- Manage the active fade state and advance it each frame based on elapsed ticks
- Interpolate transparency from initial to final values over a specified period
- Apply color-table transformations (tint, dodge, burn, negate, randomize, soft-tint)
- Blend environmental effects (water/lava/sewage/goo tints) with main fades
- Handle fade priorities and prevent rapid fade restarts
- Integrate with OpenGL fader queue when available
- Parse and apply XML-based fade configuration overrides
- Apply gamma correction to color tables

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fade_proc` | typedef | Function pointer for color-table transformation callbacks |
| `fade_definition` | struct | Complete fade spec: function, color, initial/final transparency, period, priority, flags |
| `fade_effect_definition` | struct | Environmental effect binding: fade type (e.g., water tint) and opacity |
| `fade_data` | struct | Runtime state: active flag, type, effect type, tick counter, original/animated color tables |
| `OGL_Fader` | struct (external) | OpenGL fader entry: type and 4-channel color (RGBA) |
| `XML_FaderParser` | class | XML element parser for fader definitions (index, type, opacity, period, flags, priority) |
| `XML_LiquidFaderParser` | class | XML element parser for liquid/effect fade definitions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fade` | `fade_data*` | static | Pointer to the single active fade; NULL if no fade running |
| `fades_random_seed` | `uint16` | static | LFSR seed for randomize_color_table() |
| `CurrentOGLFader` | `OGL_Fader*` | static | Pointer to current OpenGL fader queue entry being configured; NULL if OpenGL inactive |
| `FadeEffectDelay` | `int` | static | Countdown to delay fade-effect updates (MacOS workaround) |
| `fade_definitions[NUMBER_OF_FADE_TYPES]` | `fade_definition[]` | static | Array of 35+ fade type definitions (cinematic, damage flashes, color tints) |
| `fade_effect_definitions[NUMBER_OF_FADE_EFFECT_TYPES]` | `fade_effect_definition[]` | static | 5 environmental effect mappings (water, lava, sewage, jjaro, goo) |
| `actual_gamma_values[NUMBER_OF_GAMMA_LEVELS]` | `float[]` | static | Gamma correction exponents (1.3 to 0.70) |
| `original_fade_definitions` | `fade_definition*` | static | Backup for XML reset; allocated on first XML parse |
| `original_fade_effect_definitions` | `fade_effect_definition*` | static | Backup for XML reset; allocated on first XML parse |
| `FaderParser`, `LiquidFaderParser`, `FadersParser` | `XML_*Parser` | static | XML parser instances for fade configuration |

## Key Functions / Methods

### initialize_fades
- **Signature:** `void initialize_fades()`
- **Purpose:** Allocate and initialize the single global fade state at engine startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `fade` struct; clears active flag and effect type.
- **Calls:** `new` (heap allocation)
- **Notes:** Called once during engine init. Asserts on allocation failure.

### update_fades
- **Signature:** `bool update_fades()`
- **Purpose:** Advance active fade by one frame; interpolate transparency and update display if fade is active.
- **Inputs:** None (uses global `fade`, `machine_tick_count()`)
- **Outputs/Return:** `true` if fade is still active; `false` if fade finished or none active.
- **Side effects:** Modifies `fade->last_update_tick` if fade completes; calls `recalculate_and_display_color_table()`.
- **Calls:** `get_fade_definition()`, `machine_tick_count()`, `recalculate_and_display_color_table()`
- **Notes:** Handles random transparency variation if `_random_transparency_flag` is set. Returns false after final frame.

### start_fade
- **Signature:** `void start_fade(short type)`
- **Purpose:** Initiate a fade of the specified type using global world and visible color tables.
- **Inputs:** `type` ΓÇô fade type enum (0ΓÇô34)
- **Outputs/Return:** None
- **Side effects:** Delegates to `explicit_start_fade()`.
- **Calls:** `explicit_start_fade()`

### explicit_start_fade
- **Signature:** `void explicit_start_fade(short type, color_table *original, color_table *animated)`
- **Purpose:** Start a fade with explicit source/destination color tables; handle priority and restart throttling.
- **Inputs:** `type` (fade enum), `original_color_table`, `animated_color_table`
- **Outputs/Return:** None
- **Side effects:** Activates fade or interrupts existing one; calls `recalculate_and_display_color_table()` immediately.
- **Calls:** `get_fade_definition()`, `machine_tick_count()`, `recalculate_and_display_color_table()`
- **Notes:** Respects fade priority (higher interrupts lower). Prevents same fade restart within `MINIMUM_FADE_RESTART` ticks. Sets initial transparency instantly, then schedules frame-by-frame updates.

### stop_fade
- **Signature:** `void stop_fade()`
- **Purpose:** Immediately halt the active fade and apply the final transparency value.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deactivates `fade`; calls `recalculate_and_display_color_table()` with final transparency.
- **Calls:** `get_fade_definition()`, `recalculate_and_display_color_table()`

### set_fade_effect
- **Signature:** `void set_fade_effect(short type)`
- **Purpose:** Apply or remove an environmental fade effect (water/lava/sewage/etc.). Respects `FadeEffectDelay`.
- **Inputs:** `type` ΓÇô effect type enum (0ΓÇô4) or NONE to clear
- **Outputs/Return:** None
- **Side effects:** Updates `fade->fade_effect_type`; calls `recalculate_and_display_color_table()` if no main fade is active; may call `animate_screen_clut()` or `SetOGLFader()`.
- **Calls:** `SetOGLFader()`, `animate_screen_clut()`, `recalculate_and_display_color_table()`
- **Notes:** Decrement and check `FadeEffectDelay` before updating effect. Only updates display if no fade is running.

### full_fade
- **Signature:** `void full_fade(short type, color_table *original_color_table)`
- **Purpose:** Execute a fade synchronously (blocking); does not return until fade completes.
- **Inputs:** `type` (fade enum), `original_color_table`
- **Outputs/Return:** None
- **Side effects:** Allocates temporary animated color table; calls `update_fades()` in a loop; calls `Music::instance()->Idle()` each iteration.
- **Calls:** `obj_copy()`, `explicit_start_fade()`, `update_fades()`, `Music::instance()->Idle()`
- **Notes:** Used for cinematic or blocking effects. Polls `update_fades()` until it returns false.

### gamma_correct_color_table
- **Signature:** `void gamma_correct_color_table(color_table *uncorrected, color_table *corrected, short gamma_level)`
- **Purpose:** Apply gamma correction to a color table using a power function.
- **Inputs:** `uncorrected_color_table`, destination `corrected_color_table`, `gamma_level` (0ΓÇô7)
- **Outputs/Return:** None (fills `corrected_color_table` in place)
- **Side effects:** Modifies `corrected_color_table`; allocates none.
- **Calls:** `pow()` (math.h)
- **Notes:** Applies `pow(component/65535.0, gamma) * 65535.0` to each R, G, B. Asserts gamma_level is in bounds.

### recalculate_and_display_color_table
- **Signature:** `void recalculate_and_display_color_table(short type, _fixed transparency, color_table *original, color_table *animated)`
- **Purpose:** Apply fade effect and optional environmental effect to color table; update display.
- **Inputs:** `type` (main fade type or NONE), `transparency` (0ΓÇôFIXED_ONE), color tables
- **Outputs/Return:** None
- **Side effects:** Modifies `animated_color_table`; calls effect procs; updates screen palette via `animate_screen_clut()` or OpenGL fader queue.
- **Calls:** `SetOGLFader()`, `get_fade_effect_definition()`, `get_fade_definition()`, effect proc (tint/randomize/etc.), `animate_screen_clut()`, `OGL_FaderActive()`
- **Notes:** Applies environmental effect first (if active), then main fade. Sets up two OpenGL fader queue entries.

### Color-transformation functions (tint_color_table, randomize_color_table, negate_color_table, dodge_color_table, burn_color_table, soft_tint_color_table)
- **Signature:** `static void <name>(color_table *original, color_table *animated, rgb_color *color, _fixed transparency)`
- **Purpose:** Apply a specific color-blending algorithm to each entry in a color table.
- **Inputs:** Original palette, destination palette, tint color, transparency/opacity blend factor
- **Outputs/Return:** None (fills `animated`)
- **Side effects:** Each checks for `CurrentOGLFader` and sets its Type and Color fields; if OpenGL is active, returns early. Otherwise, iterates palette entries and applies pixel-wise math.
- **Calls:** Color manipulation helpers (CEILING, FLOOR, PIN, FADES_RANDOM, bit shifts, pow).
- **Notes:** Fixed-point math with shifts by `FIXED_FRACTIONAL_BITS`. Each implements a different Photoshop-style blend mode.

### SetOGLFader
- **Signature:** `void SetOGLFader(int Index)` (Index = FaderQueue_Liquid or FaderQueue_Other)
- **Purpose:** Retrieve OpenGL fader queue entry and initialize it (set Type to NONE).
- **Inputs:** Queue index (0 or 1)
- **Outputs/Return:** None
- **Side effects:** Sets `CurrentOGLFader` to the queue entry (or NULL if OpenGL inactive).
- **Calls:** `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`

### TranslateToOGLFader
- **Signature:** `static void TranslateToOGLFader(rgb_color &Color, _fixed Opacity)`
- **Purpose:** Convert 16-bit RGB color and fixed-point opacity to OpenGL's 4-channel float format.
- **Inputs:** RGB color, opacity (0ΓÇôFIXED_ONE)
- **Outputs/Return:** None (fills `CurrentOGLFader->Color[0..3]`)
- **Side effects:** Updates `CurrentOGLFader` (assumes non-NULL).
- **Calls:** None
- **Notes:** Divides by `FIXED_ONE-1` for RGB and `FIXED_ONE` for alpha. Asserts CurrentOGLFader is set.

### Fades_GetParser
- **Signature:** `XML_ElementParser *Fades_GetParser()`
- **Purpose:** Return the root XML parser for fade configuration (faders element).
- **Inputs:** None
- **Outputs/Return:** Pointer to `FadersParser`
- **Side effects:** Registers child parsers (FaderParser, LiquidFaderParser, Color parser).
- **Calls:** `AddChild()` on parser instances

---

## Control Flow Notes

**Initialization:** `initialize_fades()` is called at engine startup to allocate the global `fade` struct.

**Per-frame update:** The game loop calls `update_fades()` each frame to advance any active fade. If the fade period has elapsed, it sets final transparency and deactivates. Otherwise, it interpolates transparency and calls `recalculate_and_display_color_table()` to regenerate the animated color table and update the screen.

**Fade initiation:** Events (player damage, teleport, entering liquid, etc.) call `start_fade()` or `explicit_start_fade()` with the desired fade type. The priority system ensures high-priority fades (e.g., cinematic) are not interrupted by low-priority ones.

**Environmental effects:** `set_fade_effect()` can be called at any time to apply a liquid tint (water, lava, etc.). If no main fade is active, it immediately applies the effect; if a fade is active, the effect is queued and applied on the next `update_fades()` call.

**Color transformation:** Both `update_fades()` and direct effect changes flow through `recalculate_and_display_color_table()`, which:
1. Optionally applies the environmental effect (e.g., water tint) to the original palette.
2. Applies the main fade effect (e.g., cinematic fade-in) to the result.
3. Updates the screen via `animate_screen_clut()` (software) or OpenGL fader queue (hardware).

**Blocking fade:** `full_fade()` loops `update_fades()` until completion, pausing music between iterations.

**XML configuration:** At runtime, `Faders_GetParser()` returns parsers that allow game data files to override fade definitions (callback type, colors, timing, priority).

## External Dependencies

**Notable includes:**
- `cseries.h` ΓÇô engine base types, utility macros (PIN, MAX, CEILING, FLOOR, obj_copy)
- `fades.h` ΓÇô fade type enums, function prototypes
- `screen.h` ΓÇô `world_color_table`, `visible_color_table`, `animate_screen_clut()`
- `ColorParser.h` ΓÇô `Color_GetParser()`, `Color_SetArray()`
- `OGL_Faders.h` ΓÇô `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`, `OGL_Fader` struct
- `Music.h` ΓÇô `Music::instance()->Idle()`
- Standard C/C++: `<string.h>`, `<stdlib.h>`, `<math.h>`, `<limits.h>`

**External symbols used (defined elsewhere):**
- `machine_tick_count()` ΓÇô global tick counter
- `GetMemberWithBounds()` ΓÇô bounds-checked array accessor macro
- `_fixed`, `FIXED_ONE`, `FIXED_ONE_HALF`, `FIXED_FRACTIONAL_BITS` ΓÇô fixed-point types/constants
- `rgb_color`, `color_table` ΓÇô palette data structures
- `MACHINE_TICKS_PER_SECOND` ΓÇô timing constant
- XML parser base class (`XML_ElementParser`) and utilities
