# Source_Files/RenderOther/HUDRenderer.cpp

## File Purpose
Implements the HUD (Heads-Up Display) rendering layer for the Aleph One game engine. Manages frame-by-frame updates and rendering of player interface elements including energy/oxygen bars, weapon panels, ammunition displays, inventory screens, and network status information.

## Core Responsibilities
- Coordinate all HUD element updates via `update_everything()` each frame
- Manage shield energy and oxygen bar rendering with multi-tier display states
- Render weapon panel graphics and display current weapon name
- Draw ammunition counters (both bullet grids and energy bars)
- Display inventory/statistics with network game support (pings, rankings, kill limits)
- Implement dirty-flag pattern to optimize redrawing (only update changed elements)
- Provide motion sensor and network compass rendering hooks (called by subclasses)
- Calculate screen rectangles for HUD layout positioning

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_rectangle` | struct (from interface.h) | Rectangular region for rendering UI elements |
| `weapon_interface_data` | struct | Configuration for weapon panel display (shapes, positions, ammo data) |
| `weapon_interface_ammo_data` | struct | Ammunition display configuration (grid layout, bullet shapes, fill colors) |
| `interface_state_data` | struct | Dirty flags tracking which UI elements require redrawing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_state` | `interface_state_data` | global | Tracks dirty flags for shield, oxygen, weapon, ammo UI |
| `weapon_interface_definitions[]` | `weapon_interface_data[]` | global | Weapon UI configuration (array indexed by weapon ID) |
| `delay_time` | `short` | static (in `update_suit_oxygen`) | Throttles oxygen bar redraw frequency |
| `current_player` | `player_data*` | global (inferred) | Active player data |
| `current_player_index` | `short` | global (inferred) | Active player index |
| `dynamic_world` | game state struct | global (inferred) | Game world including player count, game info |

## Key Functions / Methods

### update_everything
- **Signature:** `bool update_everything(short time_elapsed)`
- **Purpose:** Main frame-update entry point; coordinates all HUD element updates
- **Inputs:** `time_elapsed` (ms since last frame, or `NONE` to force full redraw)
- **Outputs/Return:** `bool ForceUpdate` (true if any element was redrawn)
- **Side effects:** Clears `ForceUpdate` flag, calls all update methods, may modify `interface_state` dirty flags
- **Calls:** `update_motion_sensor()`, `update_inventory_panel()`, `update_weapon_panel()`, `update_ammo_display()`, `update_suit_energy()`, `update_suit_oxygen()`, `draw_message_area()`
- **Notes:** If Lua texture palette is active, renders debug palette instead of normal HUD; Lua palette mode forces redraw every frame

### update_suit_energy
- **Signature:** `void update_suit_energy(short time_elapsed)`
- **Purpose:** Render player shield/suit energy bar with multi-tier backgrounds (empty/single/double/triple)
- **Inputs:** `time_elapsed` (NONE forces redraw)
- **Outputs/Return:** None
- **Side effects:** Modifies `ForceUpdate`, clears `shield_is_dirty` flag, calls `draw_bar()`
- **Calls:** `get_interface_rectangle()`, `draw_bar()`, `BUILD_DESCRIPTOR()`
- **Notes:** Calculates actual bar width as proportion of max energy; uses modulo arithmetic to handle energy overage tiers

### update_suit_oxygen
- **Signature:** `void update_suit_oxygen(short time_elapsed)`
- **Purpose:** Render oxygen bar with throttled redraw rate
- **Inputs:** `time_elapsed` (NONE forces redraw)
- **Outputs/Return:** None
- **Side effects:** Maintains static `delay_time` counter; modifies `ForceUpdate`, clears `oxygen_is_dirty`
- **Calls:** `get_interface_rectangle()`, `draw_bar()`, `MIN()` macro
- **Notes:** Uses delay counter to limit redraw frequency to 2 ticks per second

### update_weapon_panel
- **Signature:** `void update_weapon_panel(bool force_redraw)`
- **Purpose:** Render current weapon graphic and name with multi-weapon variant support
- **Inputs:** `force_redraw` (true forces update regardless of dirty flag)
- **Outputs/Return:** None
- **Side effects:** Modifies `ForceUpdate`, clears `weapon_is_dirty`, sets `ammo_is_dirty`
- **Calls:** `FillRect()`, `DrawShapeAtXY()`, `getcstr()`, `get_item_name()`, `DrawText()`
- **Notes:** Complex branching for multi-weapon items (e.g., shotgun single/double); weapon name text is centered in configurable rectangle

### draw_ammo_display_in_panel
- **Signature:** `void draw_ammo_display_in_panel(short trigger_id)`
- **Purpose:** Render ammunition counter for primary or secondary weapon trigger
- **Inputs:** `trigger_id` (`_primary_interface_ammo` or `_secondary_interface_ammo`)
- **Outputs/Return:** None
- **Side effects:** Calls drawing functions; modifies on-screen pixels
- **Calls:** `get_player_desired_weapon()`, `get_player_weapon_ammo_count()`, `PIN()`, `FrameRect()`, `FillRect()`, `DrawShapeAtXY()`, `DrawShape()`, `offset_rect()`
- **Notes:** Branches on ammo type: energy weapons use vertical fill bar; bullet weapons use grid layout with optional right-to-left depletion

### update_inventory_panel
- **Signature:** `void update_inventory_panel(bool force_redraw)`
- **Purpose:** Render inventory, network statistics, or ping display
- **Inputs:** `force_redraw` (true forces update)
- **Outputs/Return:** None
- **Side effects:** Modifies `ForceUpdate`, clears `inventory_dirty` flag, may call network display functions
- **Calls:** `calculate_player_item_array()`, `draw_inventory_header()`, `FillRect()`, `NetDisplayPings()`, `get_header_name()`, `calculate_player_rankings()`, `draw_inventory_item()`, network display helpers
- **Notes:** Conditional rendering: normal inventory, network pings with latency, or game statistics (time remaining, kill limits)

### draw_bar
- **Signature:** `void draw_bar(screen_rectangle *rectangle, short width, shape_descriptor top_piece, shape_descriptor full_bar, shape_descriptor background_texture)`
- **Purpose:** Generic bar-drawing utility for energy and oxygen displays
- **Inputs:** Rectangle bounds, fill width (in pixels), three shape descriptors (cap, fill, empty)
- **Outputs/Return:** None
- **Side effects:** Renders to screen buffer
- **Calls:** `DrawShape()`, `DrawShapeAtXY()`, `offset_rect()`
- **Notes:** Handles edge cases: when width is very small, crops top cap; uses source rectangle offsets for tile-friendly rendering

## Control Flow Notes
Called once per game frame. Entry point is `update_everything()`, which acts as coordinator:
1. Resets `ForceUpdate` flag
2. Checks Lua script debug mode; if active, renders texture palette (alternative mode)
3. Otherwise updates standard HUD in sequence: motion sensor, inventory, weapons, ammo, energy, oxygen
4. Draws network message area if multiplayer
5. Returns flag indicating whether any redrawing occurred

Uses a **dirty-flag optimization** pattern: `interface_state` tracks which elements changed; update methods only redraw if dirty or `time_elapsed == NONE` (force flag).

## External Dependencies
- **Includes:** `HUDRenderer.h` (class definition), `network.h` (NetDisplayPings, NetGetLatency), `lua_script.h` (Lua texture palette API)
- **Global symbols (defined elsewhere):** `current_player`, `current_player_index`, `dynamic_world`, `weapon_interface_definitions[]`, `interface_state`, `temporary` (working buffer), shape/font/color constants (`_interface_font`, `_inventory_text_color`, etc.)
- **Virtual methods called (implemented by subclasses):** `DrawShape()`, `DrawShapeAtXY()`, `DrawText()`, `FillRect()`, `FrameRect()`, `DrawTexture()`, `update_motion_sensor()`, `render_motion_sensor()`, `SetClipPlane()`, `DisableClipPlane()`
