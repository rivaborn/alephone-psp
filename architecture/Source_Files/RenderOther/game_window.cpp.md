# Source_Files/RenderOther/game_window.cpp

## File Purpose
Manages the game window's HUD (Heads-Up Display) rendering, interface state, and weapon/ammo configuration. Handles HUD initialization, dirty-flag tracking, inventory scrolling, and XML-based customization of weapon display layouts.

## Core Responsibilities
- Initialize game window and motion sensor on startup
- Draw the HUD frame and validate world window
- Update HUD elements conditionally based on dirty flags (weapon, ammo, shield, oxygen, inventory)
- Manage inventory screen scrolling and current screen state
- Parse and apply XML configuration for weapon and ammo display layouts
- Allocate and manage SDL surface for HUD buffering
- Track and set microphone recording state for network play

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `interface_state_data` | struct | Tracks dirty flags for weapon, ammo, shield, oxygen, and inventory state |
| `weapon_interface_data` | struct | Per-weapon display configuration: panel shape, text bounds, ammo positions, multi-weapon data |
| `weapon_interface_ammo_data` | struct | Ammo display parameters: screen position, grid layout, bullet/empty shapes, rendering direction |
| `HUD_SW_Class` | class | Software renderer for HUD elements; inherits from `HUD_Class` |
| `XML_VidmasterParser` | class | Parses `<vidmaster>` tags to set achievement/cheat string set |
| `XML_AmmoDisplayParser` | class | Parses `<ammo>` tags to customize ammo display for a specific weapon slot |
| `XML_WeaponDisplayParser` | class | Parses `<weapon>` tags to customize weapon panel and ammo layouts |
| `XML_InterfaceParser` | class | Root parser for `<interface>` tag; aggregates weapon, rectangle, color, font, and vidmaster parsers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MotionSensorActive` | bool | global | Whether motion sensor display is enabled (LP addition) |
| `interface_state` | `interface_state_data` | global | Current dirty flags and interface state |
| `weapon_interface_definitions[]` | array (10 items) of `weapon_interface_data` | global | Hardcoded weapon display layouts for 10 weapons (knife, magnum, plasma pistol, etc.) |
| `OGL_HUDActive` | bool | global | Whether OpenGL rendering is active; if true, skip software HUD updates |
| `HUD_SW` | `HUD_SW_Class` | static | Software HUD renderer instance |
| `original_weapon_interface_definitions` | pointer | static | Backup of original weapon defs for XML reset |
| `VidmasterParser` | `XML_VidmasterParser` | static | Parser instance for vidmaster tag |
| `AmmoDisplayParser` | `XML_AmmoDisplayParser` | static | Parser instance for ammo tag |
| `WeaponDisplayParser` | `XML_WeaponDisplayParser` | static | Parser instance for weapon tag |
| `InterfaceParser` | `XML_InterfaceParser` | static | Root parser instance for interface tag |
| `HUD_Buffer` (extern) | `SDL_Surface*` | global | SDL surface for buffered HUD; allocated in `ensure_HUD_buffer()` |

## Key Functions / Methods

### initialize_game_window
- **Signature:** `void initialize_game_window(void)`
- **Purpose:** Set up the game window and motion sensor on engine start.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `initialize_motion_sensor()` with resource descriptors for sensor graphics.
- **Calls:** `initialize_motion_sensor()`.
- **Notes:** Entry point for HUD initialization; called once at game startup.

### draw_interface
- **Signature:** `void draw_interface(void)`
- **Purpose:** Render the entire interface frame and validate the game world window.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Early-exit if OpenGL HUD is active; calls `draw_panels()` and `validate_world_window()`.
- **Calls:** `draw_panels()`, `validate_world_window()`, `game_window_is_full_screen()`.
- **Notes:** Skipped entirely if `OGL_HUDActive` is true. Does not render in fullscreen mode.

### update_interface
- **Signature:** `void update_interface(short time_elapsed)`
- **Purpose:** Update HUD elements conditionally; redraw all if `time_elapsed == NONE`.
- **Inputs:** `time_elapsed` ΓÇô elapsed ticks since last update, or `NONE` to force full redraw.
- **Outputs/Return:** None.
- **Side effects:** Updates HUD buffer, calls `HUD_SW.update_everything()`, sets port and restores it, may request HUD drawing.
- **Calls:** `reset_motion_sensor()`, `ensure_HUD_buffer()`, `_set_port_to_HUD()`, `HUD_SW.update_everything()`, `_restore_port()`, `RequestDrawingHUD()`.
- **Notes:** Early-exit if `OGL_HUDActive` (except on full redraw). Uses dirty-flag system to optimize updates.

### draw_panels
- **Signature:** `void draw_panels(void)`
- **Purpose:** Draw the static HUD background and dynamic elements to the HUD surface.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Loads static HUD picture from resources, blits to HUD_Buffer, updates dynamic elements via `HUD_SW.update_everything()`, requests HUD drawing.
- **Calls:** `ensure_HUD_buffer()`, `get_picture_resource_from_images()`, `picture_to_surface()`, `SDL_BlitSurface()`, `SDL_FillRect()`, `_set_port_to_HUD()`, `HUD_SW.update_everything()`, `_restore_port()`, `RequestDrawingHUD()`.
- **Notes:** Caches static HUD picture to avoid repeated resource loads.

### scroll_inventory
- **Signature:** `void scroll_inventory(short dy)`
- **Purpose:** Cycle inventory screens forward or backward, skipping empty categories.
- **Inputs:** `dy` ΓÇô scroll direction: positive = next, negative = previous.
- **Outputs/Return:** None.
- **Side effects:** Calls `set_current_inventory_screen()`, marks inventory dirty.
- **Calls:** `GET_CURRENT_INVENTORY_SCREEN()`, `GET_GAME_OPTIONS()`, `calculate_player_item_array()`, `set_current_inventory_screen()`, `SET_INVENTORY_DIRTY_STATE()`.
- **Notes:** Modulo wraps around inventory screens; includes network statistics screen in multiplayer.

### mark_*_as_dirty (family)
- **Functions:** `mark_weapon_display_as_dirty()`, `mark_ammo_display_as_dirty()`, `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`
- **Purpose:** Set dirty flags so corresponding HUD elements are redrawn on next update.
- **Inputs:** None (or player_index and screen for inventory variants).
- **Outputs/Return:** None.
- **Side effects:** Modifies flags in global `interface_state`.
- **Calls:** None (direct flag updates) or `set_current_inventory_screen()` for inventory variants.

### ensure_HUD_buffer
- **Signature:** `void ensure_HUD_buffer(void)`
- **Purpose:** Allocate SDL surface for HUD if not already allocated.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Creates 640├ù480 8-bit SDL surface and caches in extern `HUD_Buffer`; alerts on allocation failure.
- **Calls:** `SDL_CreateRGBSurface()`, `SDL_DisplayFormat()`, `SDL_FreeSurface()`, `alert_user()`.
- **Notes:** Safe to call multiple times; no-op if already allocated.

### Interface_GetParser
- **Signature:** `XML_ElementParser *Interface_GetParser()`
- **Purpose:** Return the root XML parser for the interface configuration hierarchy.
- **Inputs:** None.
- **Outputs/Return:** Pointer to `InterfaceParser` (static instance) with all child parsers attached.
- **Side effects:** Sets up parser tree structure on first call (or repeated calls).
- **Calls:** `WeaponDisplayParser.AddChild()`, `InterfaceParser.AddChild()`.
- **Notes:** Called by engine to parse interface MML; child parsers are `WeaponDisplayParser`, `AmmoDisplayParser`, rectangle parser, color parser, font parser, and vidmaster parser.

### XML_AmmoDisplayParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Finalize ammo display parsing and apply parsed attributes to the original weapon definition.
- **Inputs:** (Context: previously parsed attributes via `HandleAttribute()`.)
- **Outputs/Return:** `true` on success, `false` if required index is missing.
- **Side effects:** Modifies weapon interface definition in-place via `OrigAmmo` pointer.
- **Calls:** None directly; asserts `OrigAmmo` is valid.
- **Notes:** Selectively updates only the attributes that were present in XML.

### XML_WeaponDisplayParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Finalize weapon display parsing and apply attributes to the weapon definition array.
- **Inputs:** (Context: previously parsed attributes.)
- **Outputs/Return:** `true` on success, `false` if index is missing.
- **Side effects:** Updates `weapon_interface_definitions[Index]` in-place; links `AmmoDisplayParser.OrigAmmo` to ammo subarray.
- **Calls:** None directly.
- **Notes:** Requires index attribute; backs up original definitions on first parse for reset capability.

## Control Flow Notes

**Initialization phase:** `initialize_game_window()` is called once at startup to set up the motion sensor.

**Game loop:** Each frame:
1. `update_interface(time_elapsed)` is called to update HUD state and dirty flags.
2. `draw_interface()` is called to render the frame and panels (if not fullscreen and not using OpenGL HUD).
3. If OpenGL is active, `OGL_HUDActive` prevents software HUD updates.

**Configuration:** XML parsers in `Interface_GetParser()` are invoked during engine initialization to load and apply `<interface>` blocks from MML files (Marathon Markup Language). The parser tree allows nested customization of weapon layouts, ammo positions, colors, and fonts.

## External Dependencies

### Includes / Imports
- `cseries.h` ΓÇô core platform abstractions
- `HUDRenderer_SW.h` ΓÇô software HUD renderer class
- `game_window.h` ΓÇô header for this file
- `ColorParser.h`, `FontHandler.h` ΓÇô XML parsers for colors and fonts
- `screen.h`, `screen_definitions.h` ΓÇô screen management
- `shell.h`, `preferences.h` ΓÇô game shell and user preferences
- `images.h` ΓÇô image resource handling
- `network_sound.h` ΓÇô network audio
- `GL/gl.h` (conditional) ΓÇô OpenGL

### Notable Symbols Defined Elsewhere
- `interface_state_data` ΓÇô struct (defined in headers)
- `weapon_interface_data`, `weapon_interface_ammo_data` ΓÇô structs (defined in headers)
- `initialize_motion_sensor()`, `reset_motion_sensor()` ΓÇô motion sensor functions
- `draw_panels()` ΓÇô extern declaration (this file defines it)
- `validate_world_window()`, `game_window_is_full_screen()` ΓÇô screen functions
- `get_player_data()`, `calculate_player_item_array()`, `get_item_kind()` ΓÇô player/inventory functions
- `set_network_microphone_state()` ΓÇô network audio function
- `SDL_CreateRGBSurface()`, `SDL_DisplayFormat()`, `SDL_BlitSurface()`, `SDL_FillRect()` ΓÇô SDL graphics
- `HUD_Buffer` ΓÇô extern SDL surface for HUD
- `HUD_SW` ΓÇô extern HUD_SW_Class instance
- `XML_ElementParser` ΓÇô base class for all XML parsers
