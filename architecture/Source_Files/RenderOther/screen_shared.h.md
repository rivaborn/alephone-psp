# Source_Files/RenderOther/screen_shared.h

## File Purpose
Header file sharing screen rendering infrastructure between `screen.cpp` and `screen_sdl.cpp`. Defines view size configurations, screen message and HUD element management, display utilities for text/FPS/position/microphone status, and camera/zoom control for the Marathon game engine.

## Core Responsibilities
- Define view size presets with HUD/non-HUD variants for multiple resolutions (320├ù160 to 2560├ù1600)
- Manage global screen state (color tables, view data, screen mode, bit depth)
- Provide on-screen text rendering with OpenGL/SDL fallback
- Buffer and display transient messages with automatic expiration
- Manage script-driven HUD elements (custom icons, text, colors)
- Display runtime debugging information (FPS, player position, network status)
- Control overhead map zoom and player view effects (teleport, extravision, tunnel vision)
- Handle field-of-view adjustments based on game state

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ViewSizeData` | struct | Screen resolution preset with overall dimensions, main view dimensions, HUD presence, and flags |
| `ScreenMessage` | struct | Transient on-screen message with time-remaining counter and 256-char text buffer |
| `ScriptHUDElement` | struct | Custom HUD widget with 16├ù16 icon bitmap, color index, and 256-char text |
| `screen_mode_data` | struct (defined elsewhere) | Current screen mode configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `uncorrected_color_table`, `world_color_table`, `interface_color_table`, `visible_color_table` | `color_table*` | global | Color lookup tables for gamma correction and rendering |
| `world_view` | `view_data*` | global | Main camera/viewport data (position, rotation, FOV, effects) |
| `world_pixels_structure` | `bitmap_definition*` | global | Framebuffer metadata (dimensions, pixel row pointers) |
| `screen_mode` | `screen_mode_data` | static | Current screen resolution, gamma, and mode settings |
| `PrevBufferWidth`, `PrevBufferHeight`, `PrevOffsetWidth`, `PrevOffsetHeight` | `short` | static | Previous frame dimensions to detect resize events |
| `Messages[NumScreenMessages]` | `ScreenMessage[7]` | static | Ring buffer of transient on-screen messages |
| `MostRecentMessage` | `int` | static | Index of most recently added message |
| `ScriptHUDElements[]` | `ScriptHUDElement[]` | static | Array of custom HUD elements (count from `MAXIMUM_NUMBER_OF_SCRIPT_HUD_ELEMENTS`) |
| `displaying_fps`, `frame_count`, `frame_index`, `frame_ticks[64]` | bool/short/long[] | static | FPS counter state (samples 20 frames for rolling average) |
| `ShowPosition` | bool | static | Debug flag to display player coordinates/angle |
| `HUD_RenderRequest` | bool | static | Flag requesting HUD render on next frame |
| `screen_initialized` | bool | static | Initialization guard |
| `bit_depth`, `interface_bit_depth` | short | static | Color bit depths (game and UI) |
| `DisplayTextDest`, `DisplayTextFont`, `DisplayTextStyle` | `SDL_Surface*`, `font_info*`, `short` | static | Rendering context for text output |

## Key Functions / Methods

### reset_screen
- **Signature:** `void reset_screen(void)`
- **Purpose:** Reinitialize screen state when starting a new game (called after game init)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets `world_view` overhead map, terminal mode, zoom; resets field of view based on player extravision; clears all messages and HUD elements
- **Calls:** `ResetFieldOfView()`, `reset_messages()`
- **Notes:** Cribbed from `initialize_screen()` logic

### reset_messages
- **Signature:** `void reset_messages(void)`
- **Purpose:** Clear all on-screen messages and script HUD elements
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets all `Messages[].TimeRemaining` to 0; resets all `ScriptHUDElements` to default state (color=1, text empty, no icon)
- **Calls:** None
- **Notes:** Called by `reset_screen()` and initialization

### ResetFieldOfView
- **Signature:** `void ResetFieldOfView(void)`
- **Purpose:** Restore camera field of view based on player status (normal vs. extravision)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `world_view->field_of_view` and `world_view->target_field_of_view`; disables tunnel vision
- **Calls:** None
- **Notes:** Uses `current_player->extravision_duration` to determine FOV; checks `EXTRAVISION_FIELD_OF_VIEW` vs. `NORMAL_FIELD_OF_VIEW` constants

### zoom_overhead_map_in / zoom_overhead_map_out
- **Signature:** `bool zoom_overhead_map_in(void)`, `bool zoom_overhead_map_out(void)`
- **Purpose:** Increment/decrement overhead map zoom level with bounds checking
- **Inputs:** None
- **Outputs/Return:** `true` if zoom succeeded, `false` if at limit
- **Side effects:** Modifies `world_view->overhead_map_scale` if within `[OVERHEAD_MAP_MINIMUM_SCALE, OVERHEAD_MAP_MAXIMUM_SCALE]`
- **Calls:** None

### start_teleporting_effect / start_extravision_effect
- **Signature:** `void start_teleporting_effect(bool out)`, `void start_extravision_effect(bool out)`
- **Purpose:** Trigger visual effects for teleportation and extravision state changes
- **Inputs:** `out` ΓÇô direction flag (teleport in/out or enable/disable extravision)
- **Outputs/Return:** None
- **Side effects:** Calls `start_render_effect()` for teleport fold effect; sets `world_view->target_field_of_view` for extravision
- **Calls:** `View_DoFoldEffect()`, `start_render_effect()`

### game_window_is_full_screen
- **Signature:** `bool game_window_is_full_screen(void)`
- **Purpose:** Check if current screen mode is full screen (no HUD)
- **Inputs:** None
- **Outputs/Return:** `true` if HUD flag is NOT set in current view size
- **Side effects:** None
- **Calls:** Checks `ViewSizes[screen_mode.size].flags` against `_view_show_HUD`

### change_gamma_level
- **Signature:** `void change_gamma_level(short gamma_level)`
- **Purpose:** Adjust display gamma correction and update visible color table
- **Inputs:** `gamma_level` ΓÇô gamma adjustment value
- **Outputs/Return:** None
- **Side effects:** Updates `screen_mode.gamma_level`; recomputes `world_color_table`; stops fade effect; copies corrected colors to `visible_color_table`; calls screen mode change
- **Calls:** `gamma_correct_color_table()`, `stop_fade()`, `obj_copy()`, `assert_world_color_table()`, `change_screen_mode()`, `set_fade_effect()`

### DisplayText
- **Signature:** `void DisplayText(short BaseX, short BaseY, const char *Text, unsigned char r=0xff, unsigned char g=0xff, unsigned char b=0xff)`
- **Purpose:** Render a text string to framebuffer with optional OpenGL acceleration
- **Inputs:** BaseX, BaseY ΓÇô screen coordinates; Text ΓÇô string to draw; r, g, b ΓÇô RGB color (default white)
- **Outputs/Return:** None
- **Side effects:** Uses global `DisplayTextDest`, `DisplayTextFont`, `DisplayTextStyle`; may activate OpenGL rendering or SDL blitting
- **Calls:** `OGL_RenderText()`, `draw_text()` (SDL fallback renders with black shadow at +1,+1 offset)
- **Notes:** OpenGL version skipped if in terminal mode or if overhead map is active and `OGL_MapActive` is false

### update_fps_display
- **Signature:** `static void update_fps_display(SDL_Surface *s)`
- **Purpose:** Calculate rolling FPS from frame tick samples and render to screen
- **Inputs:** `s` ΓÇô framebuffer surface to draw to
- **Outputs/Return:** None
- **Side effects:** Updates `frame_ticks[]`, `frame_index`, `frame_count`; renders FPS and network latency at top-right (adjusted for console input)
- **Calls:** `SDL_GetTicks()`, `NetGetLatency()`, `GetOnScreenFont()`, `DisplayText()`
- **Notes:** Samples 20 frames; shows `--` until buffer fills; adjusts Y position if console is active

### DisplayPosition
- **Signature:** `static void DisplayPosition(SDL_Surface *s)`
- **Purpose:** Render player's world coordinates and orientation for debugging
- **Inputs:** `s` ΓÇô framebuffer surface
- **Outputs/Return:** None
- **Side effects:** Renders 6 lines at top-left: X, Y, Z (in world units), polygon index, yaw, pitch
- **Calls:** `GetOnScreenFont()`, `DisplayText()`, uses global `temporary` buffer
- **Notes:** Only renders if `ShowPosition` flag is set; angle output is in degrees

### DisplayMessages
- **Signature:** `static void DisplayMessages(SDL_Surface *s)`
- **Purpose:** Render all active on-screen messages and script HUD elements (icons + text)
- **Inputs:** `s` ΓÇô framebuffer surface
- **Outputs/Return:** None
- **Side effects:** Decrements `TimeRemaining` for each message; renders HUD icons (OpenGL textured quad or SDL surface blit) and text with per-element colors
- **Calls:** `GetOnScreenFont()`, `DisplayText()`, `_get_interface_color()`, OpenGL functions (`glTexImage2D`, `glDrawArrays`, etc.) or SDL functions
- **Notes:** Messages loop backward from most recent; HUD elements skip lines and space columns; complex OpenGL rendering path for icons with per-endianness handling

### DisplayInputLine
- **Signature:** `static void DisplayInputLine(SDL_Surface *s)`
- **Purpose:** Render active console input prompt and text at bottom of screen
- **Inputs:** `s` ΓÇô framebuffer surface
- **Outputs/Return:** None
- **Side effects:** Renders `Console::instance()->displayBuffer()` if input is active
- **Calls:** `Console::instance()`, `GetOnScreenFont()`, `DisplayText()`

### DisplayNetMicStatus
- **Signature:** `static void DisplayNetMicStatus(SDL_Surface *s)`
- **Purpose:** Show network-game microphone status (who is speaking, all/team/disabled)
- **Inputs:** `s` ΓÇô framebuffer surface
- **Outputs/Return:** None
- **Side effects:** Renders at bottom-right corner (adjusted for console); shows speaker name, team/all icon, or disabled indicator with corresponding color
- **Calls:** `current_netgame_allows_microphone()`, `get_player_data()`, `_get_interface_color()`, `GetOnScreenFont()`, `DisplayText()`, `DisplayTextWidth()`
- **Notes:** Only renders if `game_is_networked` is true; uses `dynamic_world->speaking_player_index` to identify speaker

### SetScriptHUDColor / SetScriptHUDText / SetScriptHUDIcon / SetScriptHUDSquare
- **Signature:** `void SetScriptHUDColor(int idx, int color)`, `void SetScriptHUDText(int idx, const char* text)`, `bool SetScriptHUDIcon(int idx, const char* text, size_t rem)`, `void SetScriptHUDSquare(int idx, int _color)`
- **Purpose:** Modify HUD element state (color, text, icon graphic, or solid color square)
- **Inputs:** idx ΓÇô HUD element index (modulo'd to safety); color ΓÇô interface color index; text ΓÇô text string or icon descriptor; rem ΓÇô descriptor string length
- **Outputs/Return:** `SetScriptHUDIcon` returns `bool` (success)
- **Side effects:** Updates `ScriptHUDElements[idx]` color, text, or icon bitmap
- **Calls:** `icon::parseicon()`, `icon::seticon()`, `_get_interface_color()`, `memset()`
- **Notes:** Icon setter parses a special format (color count + hex palette + character map); color square generates a 16├ù16 solid-color bitmap

### screen_printf
- **Signature:** `void screen_printf(const char *format, ...)`
- **Purpose:** Queue a printf-style message for on-screen display with 7-second lifetime
- **Inputs:** `format`, variadic args ΓÇô printf format and arguments
- **Outputs/Return:** None
- **Side effects:** Advances `MostRecentMessage` ring buffer index; sets `TimeRemaining` to 7 seconds; formats message into `Messages[].Text` with vsnprintf
- **Calls:** `vsnprintf()`
- **Notes:** Uses safe `vsnprintf` to prevent buffer overflow (max 256 chars)

## Icon Namespace Helper Functions
- `nextc()` ΓÇô Extract next character with bounds; throws on EOF
- `isadigit()` ΓÇô Check if hex digit (0ΓÇô9, AΓÇôF, aΓÇôf)
- `digit()` ΓÇô Convert hex digit char to 0ΓÇô15
- `readuc()` ΓÇô Parse two hex digits as unsigned char
- `parseicon()` ΓÇô Parse icon descriptor string: `<count><sep><color1><sep>...<colorsN><sep><16├ù16 charmap>`; returns false on parse error
- `seticon()` ΓÇô Copy icon bitmap to HUD element with palette lookup (BGRA format, endian-aware)

## Control Flow Notes
- **Init phase:** `reset_screen()` and `reset_messages()` called during game setup
- **Frame/render phase:** Display functions (`update_fps_display()`, `DisplayMessages()`, `DisplayPosition()`, `DisplayNetMicStatus()`, `DisplayInputLine()`) called during main render pass (invoked from `screen.cpp` or `screen_sdl.cpp`)
- **Runtime:** View/zoom functions (`zoom_overhead_map_in/out()`, `start_teleporting_effect()`, `ResetFieldOfView()`) called in response to game events; HUD setters called by script system or game logic

## External Dependencies
- **Includes:** `<stdarg.h>` (variadic), `snprintf.h` (safe formatting), `Console.h` (debug console), `screen_drawing.h` (drawing primitives)
- **Symbols defined elsewhere:** `world_view`, `current_player`, `current_player_index`, `dynamic_world`, `OGL_MapActive`, `OGL_IsActive()`, `OGL_RenderText()`, `gamma_correct_color_table()`, `change_screen_mode()`, `start_render_effect()`, `View_DoFoldEffect()`, `dirty_terminal_view()`, `player_in_terminal_mode()`, `get_player_data()`, `current_netgame_allows_microphone()`, `NetGetLatency()`, `GET_GAME_OPTIONS()`, `_get_interface_color()`, `GetOnScreenFont()`, `start_tunnel_vision_effect()`, SDL functions, OpenGL functions
