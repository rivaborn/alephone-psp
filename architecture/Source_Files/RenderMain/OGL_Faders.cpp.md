# Source_Files/RenderMain/OGL_Faders.cpp

## File Purpose
Implements OpenGL-based visual fade effects (color overlays, tinting, effects) rendered each frame. Supports multiple blend modes for game effects like explosions, underwater/lava tinting, damage feedback, and visual randomness.

## Core Responsibilities
- Manages a queue of fader effects (liquid environment tints and general effects)
- Implements six distinct fader blend modes using OpenGL primitives
- Renders fader overlays using vertex arrays and configurable blend functions
- Provides fader queue access and activation checks
- Handles both standard blending and logic-op color manipulation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Fader` | struct (from OGL_Faders.h) | Holds fade type ID and RGBA color values |
| `GM_Random` | struct (from Random.h) | Seeded pseudo-random generator for static noise effects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FaderQueue` | `OGL_Fader[NUMBER_OF_FADER_QUEUE_ENTRIES]` | static | Queue of active faders (liquid + other effects) |
| `FlatStaticRandom` | `GM_Random` | static | RNG instance for generating pseudo-random color noise |
| `UseFlatStatic` | bool | static | Feature flag: use alpha blending vs. XOR logic-op for randomize effect |
| `FlatStaticColor` | uint16[4] | static | Packed RGBA color for current static frame |

## Key Functions / Methods

### OGL_FaderActive
- **Signature:** `bool OGL_FaderActive()`
- **Purpose:** Check if OpenGL rendering is active and fader effects are enabled in configuration
- **Inputs:** None
- **Outputs/Return:** Boolean; true if both OpenGL and fader flag are active
- **Side effects:** None
- **Calls:** `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `TEST_FLAG()` macro
- **Notes:** Gate function preventing fader rendering if OpenGL unavailable or disabled

### GetOGL_FaderQueueEntry
- **Signature:** `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- **Purpose:** Accessor for a specific fader in the queue
- **Inputs:** Index (0 or 1, asserted)
- **Outputs/Return:** Pointer to `OGL_Fader` in static queue
- **Side effects:** Assertion on out-of-bounds
- **Calls:** None (assert only)
- **Notes:** Simple array accessor with bounds checking; two entries support liquid + other faders

### MultAlpha
- **Signature:** `inline void MultAlpha(GLfloat *InColor, GLfloat *OutColor)`
- **Purpose:** Pre-multiply RGB channels by alpha for correct alpha blending
- **Inputs:** Input RGBA color (in-place unsafe)
- **Outputs/Return:** Out-of-place modified RGBA (alpha unchanged)
- **Side effects:** Writes to OutColor
- **Calls:** None
- **Notes:** Inline utility; RGB *= Alpha, then preserve original Alpha

### ComplementColor
- **Signature:** `inline void ComplementColor(GLfloat *InColor, GLfloat *OutColor)`
- **Purpose:** Invert RGB channels (1 ΓêÆ each channel)
- **Inputs:** Input RGBA color
- **Outputs/Return:** Out-of-place inverted RGBA (alpha unchanged)
- **Side effects:** Writes to OutColor
- **Calls:** None
- **Notes:** Used by dodge and burn effects; alpha preserved

### OGL_DoFades
- **Signature:** `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- **Purpose:** Render all active faders as screen-space quads with blend operations
- **Inputs:** Screen-space rectangle (viewport bounds for overlay)
- **Outputs/Return:** Boolean; true if any faders were rendered
- **Side effects:** 
  - GL state changes: ALPHA_TEST, BLEND, TEXTURE_2D, COLOR_LOGIC_OP, blend functions
  - Vertex rendering via `glDrawArrays()`
  - Static random state mutation (`FlatStaticRandom`)
- **Calls:** 
  - OpenGL: `glVertexPointer`, `glDisable`, `glEnable`, `glColor*`, `glDrawArrays`, `glBlendFunc`, `glLogicOp`
  - Helpers: `MultAlpha()`, `ComplementColor()`
  - Random: `FlatStaticRandom.KISS()`, `FlatStaticRandom.LFIB4()`
- **Notes:**
  - Iterates fader queue; skips `Type == NONE`
  - **Tint**: Direct color overlay
  - **Randomize**: Either flat-static (alpha-blended noise) or XOR logic-op noise
  - **Negate**: Approximates color inversion with blend `GL_ONE_MINUS_DST_COLOR`
  - **Dodge**: Color lighten using complemented color + two-pass blend
  - **Burn**: Color darken using two-pass blend with complement
  - **Soft tint**: Multiplicative blend (simulates colored light)
  - Reverts GL state after each fader

## Control Flow Notes
- **Render phase entry**: Called from main OpenGL rendering loop to overlay effects
- **Per-frame execution**: Processes fader queue once per frame, blending effects sequentially
- **State restoration**: Each fader reverts GL blend function to default (`GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`)
- **Early exit**: Returns false if faders inactive, preventing further processing

## External Dependencies
- **OpenGL**: `GL/gl.h` (platform-conditional includes for Windows, macOS, Linux)
- **Engine faders**: `fades.h` (fade type enums, fader queue constants)
- **Rendering**: `render.h`, `OGL_Render.h` (context checks, configuration access)
- **Random**: `Random.h` (GM_Random pseudo-RNG)
- **Utilities**: `cseries.h` (macros, type definitions), `OGL_Faders.h` (header)
- **Defined elsewhere**: `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `TEST_FLAG()` macro
