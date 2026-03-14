# Source_Files/RenderOther/ViewControl.cpp

## File Purpose
Implements view controller for the Aleph One game engine, managing camera field-of-view (FOV), display effects (fold, static, teleport), landscape texture rendering options, and on-screen font settings. Includes XML parsing for game-developer-friendly configuration of these parameters.

## Core Responsibilities
- Manage view effect toggles (overhead map, fold/static effects, teleport effects)
- Control and interpolate field-of-view (FOV) between normal, tunnel vision, and extra-vision states
- Store and retrieve landscape texture configuration per collection/frame
- Parse and validate XML configuration for view and landscape settings
- Provide on-screen font initialization and access

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `view_settings_definition` | struct | Boolean toggles for map, fold, static, and teleport effects |
| `FOV_settings_definition` | struct | FOV angles (normal, extra, tunnel), change rate, and aspect ratio mode |
| `LandscapeOptionsEntry` | struct | Wrapper pairing frame index with landscape texture options |
| `XML_FOVParser` | class | Parses `<fov>` elements; backs up and restores FOV on reset |
| `XML_ViewParser` | class | Parses `<view>` elements; backs up and restores view settings on reset |
| `XML_LandscapeParser` | class | Parses `<landscape>` elements; adds/updates entries in collection lists |
| `XML_LO_ClearParser` | class | Parses `<clear>` elements; clears landscape data for a collection or all |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `view_settings` | `view_settings_definition` | global | Active view effect toggles (overhead map, fold, static, teleport effects) |
| `FOV_settings` | `FOV_settings_definition` | global | Active FOV parameters and interpolation rate |
| `OnScreenFont` | `FontSpecifier` | static | On-screen HUD/debug text font ("Monaco", 12pt) |
| `ScreenFontInited` | bool | static | Tracks whether `OnScreenFont` has been initialized |
| `original_FOV_settings` | `FOV_settings_definition*` | static | Backup of FOV at XML parse start (for `ResetValues()`) |
| `original_view_settings` | `view_settings_definition*` | static | Backup of view settings at XML parse start (for `ResetValues()`) |
| `DefaultLandscape` | `LandscapeOptions` | static | Factory-default landscape options; returned when no match found |
| `LOList[NUMBER_OF_COLLECTIONS]` | `vector<LandscapeOptionsEntry>[]` | static | Per-collection storage of landscape options, indexed by collection ID |
| `FOVParser`, `ViewParser`, `LandscapeParser`, `LO_ClearParser`, `LandscapesParser` | XML parser instances | static | Singleton XML element parsers for configuration |

## Key Functions / Methods

### View_AdjustFOV
- **Signature:** `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- **Purpose:** Interpolate a FOV value toward a target over time, used to smooth FOV transitions when entering tunnel vision or extraversion states.
- **Inputs:** Current FOV value (by ref), target FOV value.
- **Outputs/Return:** Modified FOV by reference; returns `true` if FOV changed, `false` if already at target.
- **Side effects:** Modifies the passed FOV reference; reads `FOV_ChangeRate` global.
- **Calls:** `MAX()`, `MIN()` macros.
- **Notes:** Ensures `FOV_ChangeRate` is positive; clamps result to target to avoid oscillation.

### View_GetLandscapeOptions
- **Signature:** `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- **Purpose:** Look up landscape texture parameters for a given shape (collection + frame), used by render pipeline to configure background texture scaling/rotation.
- **Inputs:** Shape descriptor (encodes collection ID and frame index).
- **Outputs/Return:** Pointer to matching `LandscapeOptions`; returns `&DefaultLandscape` if no entry found.
- **Side effects:** None.
- **Calls:** `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()` macros; vector iteration.
- **Notes:** Matches on frame *or* frame `AnyFrame` sentinel; returns first match.

### LODelete / LODeleteAll
- **Signature:** `static void LODelete(int c)` / `static void LODeleteAll()`
- **Purpose:** Clear landscape options for a collection or all collections, used during XML parsing resets.
- **Inputs:** Collection ID (LODelete only).
- **Outputs/Return:** None.
- **Side effects:** Clears vector at `LOList[c]` or all vectors.
- **Calls:** `vector::clear()`.

### GetOnScreenFont
- **Signature:** `FontSpecifier& GetOnScreenFont()`
- **Purpose:** Return the on-screen HUD font, initializing it on first call.
- **Inputs:** None.
- **Outputs/Return:** Reference to static `OnScreenFont`.
- **Side effects:** Calls `OnScreenFont.Init()` and sets `ScreenFontInited = true` on first invocation.
- **Calls:** `FontSpecifier::Init()`.
- **Notes:** Lazy initialization pattern; safe to call multiple times.

### XML Parser Methods (FOV, View, Landscape)
- **Classes:** `XML_FOVParser`, `XML_ViewParser`, `XML_LandscapeParser`, `XML_LO_ClearParser`
- **Key methods:**
  - `Start()`: Backs up current settings before parsing; initializes parser state.
  - `HandleAttribute(Tag, Value)`: Parses individual XML attributes (e.g., `normal="80.0"`, `frame="2"`); validates ranges and types.
  - `AttributesDone()`: Post-processing after all attributes parsed; for `XML_LandscapeParser`, checks for duplicates and adds/updates landscape list.
  - `ResetValues()`: Restores original settings from backup (used on parse failure).
- **Side effects:** Update global settings (`FOV_settings`, `view_settings`) or landscape list (`LOList`); allocate/free backup pointers.
- **Notes:** `XML_LandscapeParser` uses iteration to find and replace duplicate frame entries before pushing new ones.

## Control Flow Notes
This file is primarily **data management and configuration**. The XML parsers (`XML_FOVParser`, `XML_ViewParser`, `XML_LandscapeParser`) are invoked during game initialization/reload when XML config files are parsed. The accessor functions (`View_MapActive()`, `View_FOV_Normal()`, etc.) are called by the render pipeline each frame to query current settings. The `View_AdjustFOV()` function is called per-frame to smoothly interpolate FOV changes. Landscape options are queried by the landscape renderer when setting up texture transforms.

## External Dependencies
- **Standard library:** `<vector>`, `<string.h>`
- **Engine core:** `cseries.h` (macros, type defs), `world.h` (angle, world_distance types)
- **Rendering:** `ViewControl.h` (declarations); `FontHandler.h` (FontSpecifier)
- **XML parsing:** `XML_ElementParser` (base class; defined elsewhere)
- **External symbols used:**
  - `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()` macros (shape_descriptors.h)
  - `StringsEqual()`, `ReadBoundedNumericalValue()`, `ReadBooleanValueAsBool()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadInt16Value()`, `UnrecognizedTag()`, `AttribsMissing()` (XML parsing utilities, defined elsewhere)
  - `FontSpecifier::Init()`, `Font_SetArray()`, `Font_GetParser()` (font management, defined elsewhere)
