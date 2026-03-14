# Source_Files/RenderOther/overhead_map.cpp

## File Purpose
Manages the overhead/automap display rendering in the Marathon engine, including configuration of visual elements (colors, fonts, widths), XML-based settings, and dispatch to platform-specific renderers (software or OpenGL). Handles both the map display mode and automap visibility tracking.

## Core Responsibilities
- Define and maintain overhead map display configuration (polygon colors, line styles, thing types, fonts)
- Select and dispatch rendering to appropriate backend (software: QuickDraw on Mac, SDL on other platforms; OpenGL optional)
- Parse and apply XML configuration for overhead map visual settings, monster/item display assignments, and visibility flags
- Initialize and manage fonts used for map annotations and title rendering
- Reset and track automap visibility state (bit arrays for which lines and polygons are revealed)
- Support multiple overhead map display modes (normal, currently visible, all)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OvhdMap_CfgDataStruct` | struct | Main configuration holding polygon colors, line definitions, thing definitions, monster/item display assignments, player entity size, annotation/map-name fonts, path color, and visibility flags |
| `XML_LiveAssignParser` | class (XML parser) | Parses XML to assign overhead-map display types to living monster types |
| `XML_DeadAssignParser` | class (XML parser) | Parses XML to assign display types to dead monsters/items by collection type |
| `XML_OvhdMapBooleanParser` | class (XML parser) | Parses boolean on/off attributes for display flags (aliens, items, projectiles, paths) |
| `XML_LineWidthParser` | class (XML parser) | Parses XML to set line pen widths at different zoom scales |
| `XML_OvhdMapParser` | class (XML parser) | Root parser managing colors and fonts arrays, aggregating all sub-parsers |
| `OverheadMap_SDL_Class` / `OverheadMap_QD_Class` | class | Software rendering backend (platform-specific) |
| `OverheadMap_OGL_Class` | class | OpenGL rendering backend |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OvhdMap_ConfigData` | `OvhdMap_CfgDataStruct` | static | Central configuration for all overhead map display settings (colors, fonts, visibility flags) |
| `MapFontsInited` | `bool` | static | Tracks whether map fonts have been initialized |
| `OGL_MapActive` | `bool` | global | Flag set externally to switch between software and OpenGL rendering modes |
| `OverheadMap_SW` | `OverheadMap_SDL_Class` / `OverheadMap_QD_Class` | static | Software rendering instance |
| `OverheadMap_OGL` | `OverheadMap_OGL_Class` | static | OpenGL rendering instance (if `HAVE_OPENGL` defined) |
| `OverheadMapMode` | `short` | static | Current display mode: normal, currently visible, or all |
| XML parser instances | various | static | Parser singletons (`LiveAssignParser`, `DeadAssignParser`, `ShowAliensParser`, etc.) |

## Key Functions / Methods

### _render_overhead_map
- **Signature:** `void _render_overhead_map(struct overhead_map_data *data)`
- **Purpose:** Main entry point for rendering the overhead map; selects appropriate backend and delegates rendering.
- **Inputs:** `overhead_map_data` struct containing mode, scale, origin, dimensions, and draw-everything flag.
- **Outputs/Return:** None (draws to display via selected renderer).
- **Side effects:** Calls `InitMapFonts()`, modifies rendering backend's config pointer, initiates render on selected backend.
- **Calls:** `InitMapFonts()`, conditional calls to `OverheadMap_OGL.Render()` or `OverheadMap_SW.Render()`.
- **Notes:** Font initialization is done on-demand before each render to ensure fonts are ready.

### InitMapFonts
- **Signature:** `static void InitMapFonts()`
- **Purpose:** Initialize fonts for map annotations and map name on first use.
- **Inputs:** None.
- **Outputs/Return:** None (modifies static `MapFontsInited` flag and `OvhdMap_ConfigData` fonts).
- **Side effects:** Calls `Init()` on all font objects in configuration; idempotent via `MapFontsInited` flag.
- **Calls:** `annotation_definition::Fonts[].Init()`, `map_name_data.Font.Init()`.
- **Notes:** Must be called before any rendering; subsequent calls are no-ops.

### OGL_ResetMapFonts
- **Signature:** `void OGL_ResetMapFonts(bool IsStarting)`
- **Purpose:** Reset OpenGL font resources (e.g., on context loss or startup).
- **Inputs:** `IsStarting` (true on initialization, false otherwise).
- **Outputs/Return:** None.
- **Side effects:** Calls `InitMapFonts()` then `OGL_Reset(IsStarting)` on each font.
- **Calls:** `InitMapFonts()`, `FontSpecifier::OGL_Reset()` for all fonts.
- **Notes:** Only active when `HAVE_OPENGL` is defined; idempotent.

### ResetOverheadMap
- **Signature:** `void ResetOverheadMap()`
- **Purpose:** Reset automap visibility state based on current overhead map mode.
- **Inputs:** None (uses global `OverheadMapMode` and `dynamic_world`).
- **Outputs/Return:** None (clears/fills `automap_lines` and `automap_polygons` bit arrays).
- **Side effects:** Memsets automap visibility arrays to 0 or 0xff based on mode.
- **Calls:** `memset()`.
- **Notes:** Mode `OverheadMap_Normal` does nothing; `OverheadMap_CurrentlyVisible` resets visibility; `OverheadMap_All` shows everything.

### OverheadMap_GetParser
- **Signature:** `XML_ElementParser *OverheadMap_GetParser()`
- **Purpose:** Export the XML parser for overhead map configuration to the main parser system.
- **Inputs:** None.
- **Outputs/Return:** Pointer to root XML parser (`OvhdMapParser`) with all sub-parsers attached.
- **Side effects:** Attaches child parsers for live/dead assignments, visibility flags, line widths, and color/font parsers to root.
- **Calls:** `OvhdMapParser.AddChild()` for each sub-parser.
- **Notes:** Called once at startup to register overhead map configuration parsing.

## Control Flow Notes

**Initialization:** Fonts and parsers are lazily initialized on first use. `OverheadMap_GetParser()` is called during XML configuration loading to register parsing rules. `fix_missing_overhead_map_fonts()` (ifdef SDL_TTF) patches font paths after config load if needed.

**Per-frame rendering:** `_render_overhead_map()` is called during screen rendering (from `render.cpp`). It ensures fonts are ready, selects the active backend (software or OpenGL based on `OGL_MapActive`), and delegates rendering.

**Automap state reset:** `ResetOverheadMap()` is called when switching display modes or starting a new level to clear or populate the visibility bit arrays based on the chosen mode.

**XML configuration:** When a map file is loaded, the parser system calls the parser hierarchy returned by `OverheadMap_GetParser()` to populate `OvhdMap_ConfigData` with colors, fonts, visibility flags, and monster/item display assignments.

## External Dependencies

- **Notable includes:**
  - `cseries.h` ΓÇö Common series utilities (types, macros, string functions)
  - `shell.h` ΓÇö For `_get_player_color()`
  - `map.h` ΓÇö World structures (`dynamic_world`, automap arrays)
  - `monsters.h` ΓÇö Monster type enums
  - `player.h` ΓÇö Player data
  - `render.h` ΓÇö View data and rendering integration
  - `flood_map.h` ΓÇö Pathfinding (included for external use)
  - `ColorParser.h` ΓÇö XML color parsing
  - `OverheadMap_SDL.h` / `OverheadMap_QD.h` ΓÇö Platform-specific software renderers
  - `OverheadMap_OGL.h` ΓÇö OpenGL renderer
  - `XML_ElementParser.h` ΓÇö XML parsing framework

- **External symbols used (defined elsewhere):**
  - `dynamic_world` ΓÇö Global world state (map counts, automap arrays)
  - `automap_lines`, `automap_polygons` ΓÇö Visibility bit arrays from map
  - XML parsing infrastructure (`XML_ElementParser`, `Color_GetParser()`, `Font_GetParser()`)
  - Renderer backend classes (`OverheadMap_SDL_Class`, `OverheadMap_OGL_Class`, etc.)
