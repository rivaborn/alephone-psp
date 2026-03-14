# Source_Files/RenderOther/ViewControl.h

## File Purpose

Header for the view controller subsystem in the Aleph One engine. Manages field-of-view parameters, teleport transition effects, landscape rendering options, and on-screen UI font configuration. Supports runtime adjustment and XML-based configuration of all viewing parameters.

## Core Responsibilities

- **FOV management**: Accessor functions for normal, extravision, and tunnel-vision FOV values; dynamic FOV adjustment toward target values
- **View effects control**: Toggle fold-in/fold-out effects and static effects on teleportation; skip interlevel teleport effects
- **Rendering parameters**: Determine whether FOV angle is fixed horizontally or vertically
- **Landscape/texture configuration**: Supply landscape options (scaling, aspect ratio, tiling) per shape descriptor
- **On-screen UI**: Provide font specifier for overhead map and other HUD elements
- **XML configuration**: Export parsers for view settings and landscape settings to enable data-driven configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LandscapeOptions` | struct | Configures landscape texture rendering: horizontal/vertical scaling exponents, OpenGL aspect-ratio adjustment, vertical repeat vs. clamp mode, and azimuth (yaw rotation) relative to view direction |

## Global / File-Static State

None.

## Key Functions / Methods

### View_FOV_Normal, View_FOV_ExtraVision, View_FOV_TunnelVision
- **Signature:** `float View_FOV_Normal()`, `float View_FOV_ExtraVision()`, `float View_FOV_TunnelVision()`
- **Purpose:** Accessors for field-of-view values in three vision modes.
- **Inputs:** None
- **Outputs/Return:** Float FOV angle in appropriate unit
- **Side effects:** None
- **Calls:** (defined elsewhere)
- **Notes:** Used to support different player vision states (e.g., active camouflage grants wider FOV, deactivated camouflage grants tunnel vision).

### View_AdjustFOV
- **Signature:** `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- **Purpose:** Smoothly transition current FOV toward a target value over time.
- **Inputs:** Current FOV (by reference), target FOV
- **Outputs/Return:** Boolean indicating whether FOV was changed in this call
- **Side effects:** Modifies the FOV reference parameter
- **Calls:** (defined elsewhere)
- **Notes:** Likely called per frame to animate FOV transitions for visual smoothness.

### View_GetLandscapeOptions
- **Signature:** `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- **Purpose:** Retrieve landscape rendering configuration for a specific texture collection.
- **Inputs:** Shape descriptor (encodes collection, shape, and CLUT indices)
- **Outputs/Return:** Pointer to LandscapeOptions struct (or NULL if not found)
- **Side effects:** None
- **Calls:** (defined elsewhere)
- **Notes:** Allows per-texture customization of scaling and tiling behavior; supports Marathon 1 vs. Marathon 2/OO landscape compatibility modes.

### View_GetParser, Landscapes_GetParser
- **Signature:** `XML_ElementParser *View_GetParser()`, `XML_ElementParser *Landscapes_GetParser()`
- **Purpose:** Provide XML parser entry points for view and landscape configuration elements.
- **Inputs:** None
- **Outputs/Return:** XML parser object pointer
- **Side effects:** None (parser itself may modify internal state when parsing)
- **Calls:** (defined elsewhere; returns objects from XMLElementParser hierarchy)
- **Notes:** Enables data-driven configuration via external XML; called during engine initialization or config reload.

**Notes on trivial accessors:** `View_MapActive()`, `View_FOV_FixHorizontalNotVertical()`, `View_DoFoldEffect()`, `View_DoStaticEffect()`, `View_DoInterlevelTeleportInEffects()`, `View_DoInterlevelTeleportOutEffects()`, and `GetOnScreenFont()` are simple boolean/value accessors; implementation likely returns cached configuration values set during XML parsing or initialization.

## Control Flow Notes

This module fits into the **render initialization and per-frame update** phases:

- **Initialization:** XML parsers are invoked to load view and landscape settings from configuration files.
- **Per-frame update:** `View_AdjustFOV()` is called to smoothly interpolate FOV changes; effect toggles and map-visibility checks determine which HUD elements and transition effects to render.
- **Rendering:** `View_GetLandscapeOptions()` is queried when rendering landscape/sky textures to apply per-collection scaling and rotation.

## External Dependencies

- **world.h:** Provides `angle` type (int16 azimuth angles in 0ΓÇô511 range for a full circle).
- **FontHandler.h:** Provides `FontSpecifier` class for on-screen text rendering.
- **shape_descriptors.h:** Provides `shape_descriptor` type (uint16 encoding collection, shape, and CLUT).
- **XML_ElementParser.h:** Provides `XML_ElementParser` base class for declarative XML-driven configuration.
