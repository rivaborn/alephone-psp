# Source_Files/RenderOther/fades.h

## File Purpose
Defines the public interface for screen fade and tint effects in the Aleph One engine. Provides functions to manage cinematic fades (black in/out), color flashes (for damage/pickups/environmental hazards), and environmental tint overlays (underwater, lava, etc.). Supports XML-based configuration of fade parameters.

## Core Responsibilities
- Define fade type constants for cinematic and effect-based screen overlays
- Manage fade state lifecycle (start, update, query completion)
- Support environmental effect tinting (water, lava, sewage, Jjaro goo)
- Perform gamma correction on color tables
- Provide delayed fade-effect application (macOS dialog bug workaround)
- Expose XML parser for fade/tint configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Fade types enum | enum | Cinematic fades, damage/item flashes, environment tints (>30 types) |
| Effect types enum | enum | Environmental condition overlays (water, lava, sewage, Jjaro goo) |
| Fader function types enum | enum | Rendering algorithm selection (tint, randomize, negate, dodge, burn, soft_tint) |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_fades
- Signature: `void initialize_fades(void);`
- Purpose: Initialize fade subsystem at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up fade state, color tables
- Calls: (defined in fades.c)
- Notes: Must be called before any fade operations

### update_fades
- Signature: `bool update_fades(void);`
- Purpose: Advance fade animation state each frame
- Inputs: None
- Outputs/Return: `true` if a fade is in progress; `false` if idle
- Side effects: Updates internal fade timer and color blending
- Calls: (defined in fades.c)
- Notes: Called once per frame from main render loop

### start_fade / stop_fade / fade_finished
- Signature: `void start_fade(short type);` / `void stop_fade(void);` / `bool fade_finished(void);`
- Purpose: Control fade state (start a new fade, stop current fade, query if fade completed)
- Inputs: `type` is an index into fade types enum
- Outputs/Return: `fade_finished()` returns `true` if current fade animation has completed
- Side effects: Modifies internal fade state
- Notes: Only one fade can be active at a time; `start_fade()` cancels any running fade

### set_fade_effect
- Signature: `void set_fade_effect(short type);`
- Purpose: Apply environmental tint overlay (water/lava/sewage/Jjaro goo)
- Inputs: `type` is an index into effect types enum
- Outputs/Return: None
- Side effects: Modifies color table tint; subject to `SetFadeEffectDelay()` buffering
- Notes: Persistent; affects all subsequent frames until changed

### get_fade_period
- Signature: `short get_fade_period(short type);`
- Purpose: Query the duration (in frame ticks) for a given fade type
- Inputs: `type` fade type index
- Outputs/Return: Duration in game ticks
- Notes: Allows callers to predict fade completion time

### gamma_correct_color_table
- Signature: `void gamma_correct_color_table(struct color_table *uncorrected_color_table, struct color_table *corrected_color_table, short gamma_level);`
- Purpose: Apply gamma correction to a color table based on display gamma level
- Inputs: `uncorrected_color_table`, `gamma_level` (0ΓÇô7, see `DEFAULT_GAMMA_LEVEL`)
- Outputs/Return: `corrected_color_table` filled with gamma-adjusted colors
- Side effects: None
- Notes: Read-only operation; no global state modified

### explicit_start_fade / full_fade
- Signature: `void explicit_start_fade(short type, struct color_table *original_color_table, struct color_table *animated_color_table);` / `void full_fade(short type, struct color_table *original_color_table);`
- Purpose: Advanced fade control with explicit color table input (for custom fade handling)
- Inputs: `type` fade type; color tables for source and destination states
- Outputs/Return: None
- Side effects: Starts fade animation with given color tables
- Notes: `explicit_start_fade()` requires both original and animated tables; `full_fade()` derives animated table

### SetFadeEffectDelay
- Signature: `void SetFadeEffectDelay(int _FadeEffectDelay);`
- Purpose: Delay subsequent `set_fade_effect()` calls by N frames
- Inputs: Number of calls to `set_fade_effect()` to ignore
- Outputs/Return: None
- Side effects: Sets internal delay counter
- Notes: Workaround for macOS bug where dialog box clearing paints over fader effect; documented in comments

### Faders_GetParser
- Signature: `XML_ElementParser *Faders_GetParser();`
- Purpose: Return XML parser for fade/tint configuration
- Inputs: None
- Outputs/Return: Pointer to parser object (for use with XML config loader)
- Side effects: None
- Notes: Allows XML-driven fade parameter configuration (Loren Petrich addition)

## Control Flow Notes
Fades integrate into the frame-update pipeline:
- **Init**: `initialize_fades()` called during engine startup
- **Per-frame**: `update_fades()` advances fade animation; `set_fade_effect()` can be called to change environmental tint
- **Query**: `fade_finished()` allows code to detect fade completion (e.g., to trigger post-fade game state changes)
- **Render**: (Actual color table application to screen occurs in render backend, not exposed here)

Environmental tints are persistent overlays; cinematic/flash fades are temporary animations.

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇö for XML fade configuration
- Reference to `struct color_table` (defined elsewhere, likely in a graphics header)
- Implicit dependency on game tick/frame timing system (not visible in this header)
