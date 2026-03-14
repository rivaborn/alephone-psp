# Source_Files/RenderOther/screen.h

## File Purpose
Declares screen rendering, display mode, color table, and HUD management for the Aleph One game engine. Supports multiple resolutions, hardware acceleration (OpenGL), visual effects, color palettes, and scripted HUD overlays.

## Core Responsibilities
- Screen mode initialization, switching, and fullscreen toggling
- Color table (CLUT) management for world, interface, and visible palettes
- Screen rendering and frame-by-frame updates
- Visual effects (teleport, extravision)
- Overhead map display and zoom control
- HUD drawing and script-driven HUD element configuration
- Screen validation, clearing, and gamma adjustment
- Screenshot dumping and tunnel vision mode

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed) | enum | Screen sizes (e.g., `_320_160_HUD`, `_640_480`, `_1920_1200`, PSP variants) and percentage scales (`_50_percent`, `_100_percent`) |
| (unnamed) | enum | Hardware acceleration codes: `_no_acceleration`, `_opengl_acceleration` |
| color_table | struct | Color palette data (defined elsewhere) |
| screen_mode_data | struct | Screen mode configuration (defined in SHELL.H / PREFERENCES.H) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| world_color_table | struct color_table* | extern | Active world color palette |
| visible_color_table | struct color_table* | extern | Visible/rendered color palette |
| interface_color_table | struct color_table* | extern | UI/HUD color palette |

## Key Functions / Methods

### render_screen
- Signature: `void render_screen(short ticks_elapsed)`
- Purpose: Main per-frame rendering entry point
- Inputs: Elapsed ticks since last frame
- Outputs/Return: None
- Side effects: Updates display, applies color animations, draws HUD
- Calls: (Not inferable from header)
- Notes: Called once per frame during main loop

### enter_screen / exit_screen
- Signature: `void enter_screen(void)`, `void exit_screen(void)`
- Purpose: Screen subsystem lifecycle management
- Inputs/Outputs: None
- Side effects: Initialize or cleanup screen resources
- Notes: Paired calls for startup/shutdown

### initialize_screen
- Signature: `void initialize_screen(struct screen_mode_data *mode, bool ShowFreqDialog)`
- Purpose: Initialize screen with specified mode and optional monitor-frequency dialog
- Inputs: Screen mode, whether to show frequency dialog
- Outputs/Return: None
- Side effects: Sets up display and rendering context
- Notes: LP addition; called at engine startup

### change_screen_mode
- Signature: `void change_screen_mode(struct screen_mode_data *mode, bool redraw)`
- Purpose: Switch to a different screen resolution/mode
- Inputs: New screen mode, redraw flag
- Outputs/Return: None
- Side effects: Modifies display dimensions and framebuffer

### change_screen_clut / animate_screen_clut
- Signature: `void change_screen_clut(struct color_table *color_table)`, `void animate_screen_clut(struct color_table *color_table, bool full_screen)`
- Purpose: Switch active color palette or animate palette transitions
- Inputs: Color table, optionally full-screen flag
- Outputs/Return: None
- Side effects: Immediate or animated color palette swap

### start_teleporting_effect / start_extravision_effect
- Signature: `void start_teleporting_effect(bool out)`, `void start_extravision_effect(bool out)`
- Purpose: Trigger screen distortion effects for gameplay events
- Inputs: Direction flag (in/out for teleport; on/off for extravision)
- Outputs/Return: None
- Side effects: Applies visual filter or animation

**HUD / Script Functions:**
- `RequestDrawingHUD()` ΓÇô Flag HUD for next frame
- `SetScriptHUDColor(int idx, int color)` ΓÇô Set color of HUD element
- `SetScriptHUDText(int idx, const char* text)` ΓÇô Set or clear HUD text element (NULL/"" removes)
- `SetScriptHUDIcon(int idx, const char* icon, size_t length)` ΓÇô Set icon for HUD element
- `SetScriptHUDSquare(int idx, int color)` ΓÇô Draw colored square in HUD element
- `ShowMessage(char *Text)` ΓÇô Display temporary debug/info message on screen

**Utility Functions:**
- `toggle_overhead_map_display_status()` ΓÇô Show/hide map
- `zoom_overhead_map_in/out()` ΓÇô Map zoom (returns whether scale changed)
- `toggle_fullscreen(bool fs)` / `toggle_fullscreen()` ΓÇô Fullscreen toggle
- `darken_world_window()` / `validate_world_window()` ΓÇô Darken or redraw world view
- `dump_screen()` ΓÇô Save screenshot to file
- `GetTunnelVision()` / `SetTunnelVision(bool)` ΓÇô Tunnel vision effect mode
- `GetSizeWithHUD()` / `GetSizeWithoutHUD()` / `SizeWithHUD()` / `SizeWithoutHUD()` ΓÇô Map between HUD and non-HUD resolution variants

## Control Flow Notes
- **Initialization:** `initialize_screen()` ΓåÆ `enter_screen()` during startup
- **Frame loop:** `render_screen(ticks_elapsed)` called once per frame; can call color/effect animations
- **Mode changes:** `change_screen_mode()` or `toggle_fullscreen()` as needed
- **Shutdown:** `exit_screen()` ΓåÆ `reset_screen()` (CP addition for game restart)
- **HUD updates:** Script/game code calls `SetScriptHUD*()` functions; `RequestDrawingHUD()` signals `render_screen()` to draw them

## External Dependencies
- **color_table** (defined elsewhere) ΓÇô Palette data structure
- **screen_mode_data** (SHELL.H / PREFERENCES.H) ΓÇô Mode configuration
- **Rect** (QuickDraw) ΓÇô Rectangle bounds
- **OpenGL** (implicit) ΓÇô Hardware acceleration backend option
- **Pfhortran** (noted in comments) ΓÇô Script engine for HUD element control
