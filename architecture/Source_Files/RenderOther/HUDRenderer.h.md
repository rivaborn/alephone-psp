# Source_Files/RenderOther/HUDRenderer.h

## File Purpose
Defines the HUD (Heads-Up Display) rendering base class and associated data structures for the Aleph One game engine (Marathon-based). Provides constants, texture IDs, state management, and an abstract interface for rendering HUD elements like weapon panels, energy/oxygen bars, inventory, and motion sensor.

## Core Responsibilities
- Define texture IDs for all HUD visual elements (energy bars, weapon panels, ammo, motion sensor blips, etc.)
- Manage HUD dirty state tracking (inventory and interface dirtiness flags)
- Store weapon interface configuration data (panel positions, ammo display layout)
- Provide abstract base class (`HUD_Class`) for platform-specific HUD rendering implementations
- Coordinate HUD updates across suit energy, oxygen, weapons, inventory, and motion sensor
- Abstract shape/text rendering, clipping, and entity blip drawing operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `weapon_interface_ammo_data` | struct | Defines layout and visual representation of ammo/energy display in a weapon panel (position, count, bullet shape, empty state) |
| `weapon_interface_data` | struct | Complete configuration for a weapon's interface panel: panel shape, weapon name position, ammo data for primary/secondary |
| `interface_state_data` | struct | Tracks which HUD subsystems need redrawing: ammo, weapon, shield, oxygen dirtiness flags |
| `HUD_Class` | class | Abstract base class defining the HUD rendering contract; delegates to platform-specific implementations via virtual methods |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_state` | `interface_state_data` | global | Tracks which HUD components are "dirty" (need redraw) |
| `weapon_interface_definitions` | `weapon_interface_data[10]` | global | Array of weapon panel configurations (positions, shapes, ammo layout) |

## Key Functions / Methods

### update_everything
- **Signature:** `bool update_everything(short time_elapsed);`
- **Purpose:** Main HUD update function; coordinates all HUD subsystem updates.
- **Inputs:** `time_elapsed` ΓÇô ticks since last frame
- **Outputs/Return:** `bool` ΓÇô indicates whether anything was updated
- **Side effects:** Modifies `interface_state` dirty flags; may trigger redraws
- **Calls:** Protected update methods (`update_suit_energy`, `update_weapon_panel`, `update_inventory_panel`, etc.)
- **Notes:** Entry point for frame-by-frame HUD updates

### update_suit_energy / update_suit_oxygen
- **Purpose:** Update energy/oxygen bar display state (detect changes, mark as dirty)
- **Inputs:** `short time_elapsed`
- **Calls:** `draw_bar()` (if dirty)

### update_weapon_panel / update_ammo_display / update_inventory_panel
- **Purpose:** Update weapon panel, ammo counter, and inventory display state
- **Inputs:** `bool force_redraw` ΓÇô skip dirty-flag check and always redraw
- **Calls:** `draw_*` methods

### motion_sensor_scan / update_motion_sensor / render_motion_sensor
- **Purpose:** Scan for entities; update and render motion sensor display
- **Inputs:** `short ticks_elapsed` or `short time_elapsed`
- **Virtual:** `update_motion_sensor()` and `render_motion_sensor()` are pure virtual (platform-specific)

### draw_bar
- **Signature:** `void draw_bar(screen_rectangle *rectangle, short actual_height, shape_descriptor top_piece, shape_descriptor full_bar, shape_descriptor background_piece);`
- **Purpose:** Render a segmented progress bar (e.g., energy, oxygen)
- **Inputs:** `rectangle` ΓÇô destination bounds; `actual_height` ΓÇô current bar level; shape descriptors for visual components
- **Calls:** `DrawShape()` (virtual)

### Pure Virtual Methods (Rendering Contract)
- **`DrawShape(shape_descriptor, screen_rectangle *dest, screen_rectangle *src)`** ΓÇô Draw a shape with optional source clipping
- **`DrawShapeAtXY(shape_descriptor, short x, short y, bool transparency)`** ΓÇô Draw shape at screen coordinates
- **`DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`** ΓÇô Draw text string
- **`FillRect(screen_rectangle *r, short color_index)`** ΓÇô Fill rectangle with color
- **`FrameRect(screen_rectangle *r, short color_index)`** ΓÇô Draw rectangle outline
- **`DrawTexture(shape_descriptor, short x, short y, int size)`** ΓÇô Draw textured element
- **`SetClipPlane(int x, int y, int c_x, int c_y, int radius)`** ΓÇô Enable circular clipping region
- **`DisableClipPlane(void)`** ΓÇô Disable clipping
- **Purpose:** Subclasses implement these to adapt HUD rendering to different graphics backends (software, OpenGL, etc.)

## Control Flow Notes
- **Frame entry:** `update_everything()` called once per frame
- **Update phase:** Checks dirty flags and player state; updates HUD subsystems
- **Render phase:** Pure virtual methods invoked by draw routines; delegates to subclass implementations
- **Motion sensor:** Scanned/updated independently; blips drawn via `draw_entity_blip()` / `erase_entity_blip()`
- **Inventory/weapon panels:** Drawn only if marked dirty or forced

## External Dependencies
- **Notable includes:**
  - `map.h` ΓÇô Map/world geometry (polygon data)
  - `interface.h` ΓÇô High-level interface state
  - `player.h` ΓÇô Player data (energy, oxygen, items, weapons)
  - `SoundManager.h` ΓÇô Sound effects for HUD interactions
  - `motion_sensor.h` ΓÇô Motion sensor data structures
  - `items.h` ΓÇô Item/powerup definitions
  - `weapons.h` ΓÇô Weapon type definitions
  - `network_games.h` ΓÇô Multiplayer game state
  - `screen_drawing.h` ΓÇô Screen drawing utilities (screen_rectangle, etc.)
- **External symbols used:** `world_point3d`, `world_point2d`, `screen_rectangle` (defined in map.h / screen_drawing.h); shape descriptor constants; texture/shape IDs from shapes system
