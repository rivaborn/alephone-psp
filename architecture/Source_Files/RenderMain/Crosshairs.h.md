# Source_Files/RenderMain/Crosshairs.h

## File Purpose
Interface header for the crosshair rendering system in Aleph One. Defines the data structure for crosshair configuration and provides functions to render, configure, and toggle crosshairs in the game viewport.

## Core Responsibilities
- Define `CrosshairData` structure for storing crosshair visual properties (color, thickness, opacity, shape)
- Provide configuration dialog for user customization of crosshair appearance
- Manage crosshair active/inactive state
- Render crosshairs to screen via platform-specific graphics APIs (QuickDraw for Mac, SDL for cross-platform)
- Retrieve crosshair configuration from persistent preferences

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairData` | struct | Stores all crosshair visual properties: color, thickness, offset from center, length, shape type (real or circle), opacity, and pre-calculated GL color values for rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| (implicit global) | `CrosshairData` | singleton via `GetCrosshairData()` | Crosshair configuration loaded from preferences; modified via `Configure_Crosshairs()` dialog |

## Key Functions / Methods

### Configure_Crosshairs
- **Signature:** `bool Configure_Crosshairs(CrosshairData &Data)`
- **Purpose:** Display configuration dialog to allow user to customize crosshair appearance
- **Inputs:** Reference to a `CrosshairData` struct to be modified
- **Outputs/Return:** `true` if user confirmed changes, `false` if canceled; struct is only modified on `true`
- **Side effects:** May display UI dialog; preferences may be persisted afterward
- **Calls:** Implemented in `PlayerDialogs.c`

### GetCrosshairData
- **Signature:** `CrosshairData& GetCrosshairData()`
- **Purpose:** Retrieve the active crosshair configuration from persistent preferences
- **Inputs:** None
- **Outputs/Return:** Reference to the current `CrosshairData` configuration
- **Side effects:** None
- **Calls:** Implemented in `preferences.c`

### Crosshairs_IsActive / Crosshairs_SetActive
- **Purpose:** Query and toggle crosshair visibility state
- **Inputs:** `NewState` (bool) for setter
- **Outputs/Return:** Current active state (bool)
- **Side effects:** State affects whether `Crosshairs_Render()` draws anything

### Crosshairs_Render
- **Signature:** Multiple overloads (QD on Mac, SDL on other platforms)
  - Mac: `Crosshairs_Render(Rect &ViewRect)`, `Crosshairs_Render(GrafPtr Context)`, `Crosshairs_Render(GrafPtr Context, Rect &ViewRect)`
  - SDL: `Crosshairs_Render(SDL_Surface *s)`
- **Purpose:** Draw the crosshair in the center of the specified viewport
- **Inputs:** Graphics context and optional view rectangle; defaults to current context/screen bounds if omitted
- **Outputs/Return:** `bool` (success/failure)
- **Side effects:** Modifies graphics context to draw crosshair; reads active state and `CrosshairData` configuration

## Control Flow Notes
- **Initialization:** Crosshair config loaded from preferences via `GetCrosshairData()`
- **Per-frame render:** `Crosshairs_Render()` called each frame if active to draw crosshair at viewport center
- **User interaction:** `Configure_Crosshairs()` allows runtime customization of appearance
- **State management:** Active/inactive state can be toggled independently of configuration

## External Dependencies
- `RGBColor` type (defined elsewhere)
- QuickDraw types (`Rect`, `GrafPtr`) for Mac builds
- SDL types (`SDL_Surface`) for SDL builds
- `PlayerDialogs.c` ΓÇö implementation of configuration dialog
- `preferences.c` ΓÇö storage and retrieval of crosshair configuration
