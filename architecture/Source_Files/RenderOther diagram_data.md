# Source_Files/RenderOther/ChaseCam.cpp
## File Purpose
Implements a third-person chase camera system for Marathon, providing a Halo-like behind-the-back view. The camera follows the player with configurable springiness and inertia, and respects level geometry to avoid clipping through walls.

## Core Responsibilities
- Maintain chase camera state (active/inactive, reset flag)
- Store and manage camera position history for inertia calculations
- Apply spring/damping physics to smooth camera movement
- Raycast camera position against level geometry (walls, floors, ceilings)
- Provide interface for enabling/disabling camera and switching horizontal offset
- Update camera position once per game tick

## External Dependencies
- **Map geometry:** `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, geometry query functions
- **Player state:** `current_player` global (camera location, orientation, polygon)
- **Configuration:** `GetChaseCamData()` (preferences), `TEST_FLAG()` macro (bit ops)
- **Network:** `NetAllowBehindview()` (multiplayer restrictions)
- **Math/geometry:** `translate_point3d()`, `translate_point2d()`, `NORMALIZE_ANGLE()` macro
- **Includes:** `cseries.h`, `map.h`, `player.h`, `ChaseCam.h`, `network.h`, `<limits.h>`

# Source_Files/RenderOther/ChaseCam.h
## File Purpose
Interface for a third-person chase camera system that follows the player, similar to Halo's implementation. Provides functions to configure, initialize, update, and query the state of a follow camera that maintains a dynamic offset from the player's position and orientation.

## Core Responsibilities
- Define chase camera configuration parameters (distance offsets, physics tuning, visibility flags)
- Initialize and manage chase camera lifecycle (startup, per-level reset, shutdown)
- Update camera position and orientation each frame using physics simulation
- Track active/inactive state and support toggling the camera on/off
- Query final camera position and viewing angles for rendering
- Support side-switching behavior when camera is offset laterally

## External Dependencies
- **`world.h`**: Core type definitions (`world_point3d`, `angle`, `world_distance`) and coordinate system macros
- **"PlayerDialogs.c"**: Implements `Configure_ChaseCam()` (settings UI)
- **"preferences.c"**: Implements `GetChaseCamData()` (persistent configuration loading)

# Source_Files/RenderOther/computer_interface.cpp
## File Purpose
Implements the computer terminal interface system for the game engine, enabling players to interact with in-game terminals that display structured text, graphics, audio, and navigation options. Manages terminal rendering, input handling, state tracking, and serialization across save/load cycles.

## Core Responsibilities
- **Terminal Lifecycle**: Initialize terminal manager, set up per-player terminal state, enter/exit terminal mode
- **Terminal Rendering**: Draw terminal UI (borders, text, pictures), manage clipping regions, apply text styling (bold, italic, color)
- **Text Layout**: Calculate line breaks with word-wrapping, measure text width, determine text bounds for multi-line content
- **State Management**: Track per-player terminal state (current group, line, phase), handle terminal progression through content groups
- **User Input**: Process keyboard input for navigation (next/previous group, page up/down, abort)
- **Terminal Groups**: Manage different group types (logon, success, failure, information, checkpoint, sound, movie, picture, teleport, etc.)
- **Serialization**: Pack/unpack terminal data and player state for save files
- **Terminal Effects**: Handle logon animations, static displays, text encoding/decoding

## External Dependencies
- **Includes**: `cseries.h` (common utilities), `FileHandler.h` (I/O), `world.h`/`map.h`/`player.h` (game world types), `screen_drawing.h`/`screen.h` (rendering), `SoundManager.h` (audio), `lua_script.h` (scripting callbacks), `Packing.h` (binary serialization)
- **External Functions**: `_get_font_spec()`, `_get_interface_color()`, `GetInterfaceFont()`, `GetInterfaceStyle()`, `play_object_sound()`, `Sound_TerminalLogon()`, `L_Call_Terminal_Exit()`, `get_player_data()`, `draw_polygon()`, `char_width()`, `text_width()`
- **External Symbols**: `world_pixels`, `draw_surface` (SDL surfaces); `dynamic_world`, `current_player_index` (game state); `game_is_networked` (network flag)

# Source_Files/RenderOther/computer_interface.h
## File Purpose
Defines the interface for the in-game computer/terminal display system. Manages player interaction with terminals, including text rendering with formatting, input handling, viewport management, and persistence of terminal state via pack/unpack operations.

## Core Responsibilities
- Initialize and manage the terminal interface system
- Handle player entry into and exit from terminal mode
- Render formatted terminal text with style markup (bold, italic, underline)
- Process player input while interacting with terminals
- Pack and unpack terminal state for save/load operations
- Preprocess and encode terminal text resources from map data
- Manage per-player terminal viewing state (viewport bounds, offsets)

## External Dependencies
- **Includes:** Standard C headers (implied by types like `uint8`, `uint32`, `size_t`, `bool`)
- **Game systems:** Player management (indexed by `player_index`), rendering/graphics, sound and movie systems (for terminal content)
- **Conditional code:** Preprocessing functions (text parsing, PICT resources, checkpoints) gated by `PREPROCESSING_CODE` macro (used at map compile-time, not runtime)

# Source_Files/RenderOther/fades.cpp
## File Purpose
Implements the game's screen fade and color-effect system. Manages timed fade transitions (cinematic fades, damage flashes, color tints) and applies various color-manipulation effects to the game's palette. Supports XML configuration and OpenGL rendering backends.

## Core Responsibilities
- Manage the active fade state and advance it each frame based on elapsed ticks
- Interpolate transparency from initial to final values over a specified period
- Apply color-table transformations (tint, dodge, burn, negate, randomize, soft-tint)
- Blend environmental effects (water/lava/sewage/goo tints) with main fades
- Handle fade priorities and prevent rapid fade restarts
- Integrate with OpenGL fader queue when available
- Parse and apply XML-based fade configuration overrides
- Apply gamma correction to color tables

## External Dependencies

**Notable includes:**
- `cseries.h` ΓÇô engine base types, utility macros (PIN, MAX, CEILING, FLOOR, obj_copy)
- `fades.h` ΓÇô fade type enums, function prototypes
- `screen.h` ΓÇô `world_color_table`, `visible_color_table`, `animate_screen_clut()`
- `ColorParser.h` ΓÇô `Color_GetParser()`, `Color_SetArray()`
- `OGL_Faders.h` ΓÇô `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`, `OGL_Fader` struct
- `Music.h` ΓÇô `Music::instance()->Idle()`
- Standard C/C++: `<string.h>`, `<stdlib.h>`, `<math.h>`, `<limits.h>`

**External symbols used (defined elsewhere):**
- `machine_tick_count()` ΓÇô global tick counter
- `GetMemberWithBounds()` ΓÇô bounds-checked array accessor macro
- `_fixed`, `FIXED_ONE`, `FIXED_ONE_HALF`, `FIXED_FRACTIONAL_BITS` ΓÇô fixed-point types/constants
- `rgb_color`, `color_table` ΓÇô palette data structures
- `MACHINE_TICKS_PER_SECOND` ΓÇô timing constant
- XML parser base class (`XML_ElementParser`) and utilities

# Source_Files/RenderOther/fades.h
## File Purpose
Defines the public interface for screen fade and tint effects in the Aleph One engine. Provides functions to manage cinematic fades (black in/out), color flashes (for damage/pickups/environmental hazards), and environmental tint overlays (underwater, lava, etc.). Supports XML-based configuration of fade parameters.

## Core Responsibilities
- Define fade type constants for cinematic and effect-based screen overlays
- Manage fade state lifecycle (start, update, query completion)
- Support environmental effect tinting (water, lava, sewage, Jjaro goo)
- Perform gamma correction on color tables
- Provide delayed fade-effect application (macOS dialog bug workaround)
- Expose XML parser for fade/tint configuration

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇö for XML fade configuration
- Reference to `struct color_table` (defined elsewhere, likely in a graphics header)
- Implicit dependency on game tick/frame timing system (not visible in this header)

# Source_Files/RenderOther/FontHandler.cpp
## File Purpose
Implements cross-platform font handling for the Aleph One game engine, supporting MacOS Quickdraw, SDL, and OpenGL rendering. Manages font specifications (name, size, style), text metrics, and provides XML configuration for fonts with layout support (wrapping, alignment, truncation).

## Core Responsibilities
- Font specification and initialization (name sets, size, style, file paths)
- Platform-specific font loading: MacOS Quickdraw font ID resolution, SDL font resource loading
- Text metrics calculation: character widths, line spacing, ascent/descent caching
- OpenGL font texture generation: glyph-to-texture atlas layout, display list compilation
- OpenGL text rendering: character display lists, modelview transformations, state management
- Text layout: wrapping at word boundaries, horizontal/vertical alignment, truncation
- XML parsing and dynamic font reconfiguration via FontList updates

## External Dependencies
- **OpenGL:** glGenTextures(), glTexImage2D(), glBindTexture(), glCallList(), glMatrixMode(), glTranslate(), glPush/PopMatrix(), glPush/PopAttrib(), glEnable(), glDisable()
- **MacOS Quickdraw:** TextFont(), TextFace(), TextSize(), GetFNum(), GetFontInfo(), CharWidth(), DrawChar(), GetFont(), SetFont(), GWorld/PixMap management
- **SDL:** SDL_CreateRGBSurface(), SDL_FillRect(), SDL_MapRGB(), SDL_FreeSurface(), load_font(), char_width(), draw_text() [defined in screen_drawing_sdl.cpp]
- **Game framework:** shape_descriptors.h, screen_drawing.h (screen_rectangle, RECTANGLE_WIDTH/HEIGHT macros)
- **XML parsing:** XML_ElementParser (base class), StringsEqual(), ReadBoundedInt16Value(), ReadInt16Value()
- **Utilities:** FindNextName(), FindNameEnd() (static string parsing helpers for font name lists); operator==(), operator=() (FontSpecifier comparison/assignment)

# Source_Files/RenderOther/FontHandler.h
## File Purpose
Defines the `FontSpecifier` class for managing text font specifications, metrics, and rendering in the Aleph One game engine. Handles font parameter specification via XML, platform-specific font operations (macOS/SDL), and OpenGL text rendering with support for text alignment and wrapping.

## Core Responsibilities
- Manage font parameters (name set, size, style, line height adjustments)
- Compute and cache font metrics (ascent, descent, leading, character widths)
- Load and update fonts via `Update()` and XML parser integration
- Render text to screen coordinates with OpenGL (with modelview matrix manipulation)
- Draw formatted text with alignment and line wrapping (OpenGL)
- Maintain platform-specific font resources (macOS ID, SDL font_info, or OpenGL textures)
- Provide character and text width queries for layout and centering

## External Dependencies
- **cseries.h** ΓÇô Core type definitions (`uint8`, `uint32`, `int16`, etc.)
- **XML_ElementParser.h** ΓÇô Base class for XML configuration parsing
- **sdl_fonts.h** ΓÇô SDL font abstraction (`font_info` class)
- **OpenGL headers** ΓÇô Platform-specific conditional includes (`GL/gl.h` on Linux, `OpenGL/gl.h` on macOS, etc.)
- **Defined elsewhere:** `screen_rectangle` (from screen_drawing.h or similar)

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

# Source_Files/RenderOther/game_window.h
## File Purpose
Header file for game window initialization and HUD (Heads-Up Display) rendering. Declares the public interface for managing the game window, drawing UI elements, updating interface state per frame, and marking various UI components as needing redraw via a dirty-flag pattern.

## Core Responsibilities
- Game window initialization
- HUD buffer management and OpenGL rendering
- Per-frame interface updates with time-delta awareness
- Inventory UI interaction (scrolling)
- Dirty-flag marking for: ammo, shield, oxygen, weapon displays, player inventory, and network stats
- Microphone recording state for interface
- XML-based interface configuration via parser access

## External Dependencies
- **`Rect`** ΓÇö Rectangle structure (defined elsewhere, likely a standard geom type)
- **`XML_ElementParser`** ΓÇö Class for parsing interface XML definitions (forward-declared)
- **OpenGL** ΓÇö Implicit dependency via `OGL_DrawHUD` naming; rendering backend
- **Time system** ΓÇö Elapsed time tracked between frames

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

## External Dependencies
- **Includes:** `HUDRenderer.h` (class definition), `network.h` (NetDisplayPings, NetGetLatency), `lua_script.h` (Lua texture palette API)
- **Global symbols (defined elsewhere):** `current_player`, `current_player_index`, `dynamic_world`, `weapon_interface_definitions[]`, `interface_state`, `temporary` (working buffer), shape/font/color constants (`_interface_font`, `_inventory_text_color`, etc.)
- **Virtual methods called (implemented by subclasses):** `DrawShape()`, `DrawShapeAtXY()`, `DrawText()`, `FillRect()`, `FrameRect()`, `DrawTexture()`, `update_motion_sensor()`, `render_motion_sensor()`, `SetClipPlane()`, `DisableClipPlane()`

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

# Source_Files/RenderOther/HUDRenderer_OGL.cpp
## File Purpose
Implements OpenGL-based rendering of the game HUD (Heads-Up Display) for the Aleph One engine. Manages loading the static HUD backdrop texture from resources and rendering both static background and dynamic HUD elements (weapons display, ammo, shield, oxygen, motion sensor, text).

## Core Responsibilities
- Load and cache the static HUD backdrop texture on demand, tiled into 6 optimally-sized GL textures for efficient rendering
- Render the HUD background each frame and coordinate rendering of all dynamic overlay elements
- Provide drawing primitives for HUD elements: bitmapped shapes, textures, colored text, filled/framed rectangles
- Manage the motion sensor display with circular clip-plane approximation for rendering blips
- Handle platform-specific pixel format conversions (1555 ARGB on Mac, 8888 ARGB, SDL surfaces)
- Reset font and texture caches during initialization or when resources need reloading

## External Dependencies
- **FontHandler.h**: `FontSpecifier::OGL_DrawText()` for text rendering
- **game_window.h**: Dirty-marking functions (`mark_*_as_dirty()`); Mac HUD buffer pointer
- **images.h**: `get_picture_resource_from_images()`, `picture_to_surface()`
- **OpenGL**: Platform-specific includes (`<GL/gl.h>`)
- **SDL** (conditional): `SDL_Surface`, `SDL_BlitSurface()` for non-Mac rendering
- **MacOS APIs** (conditional): `GWorldPtr`, `PixMapHandle`, pixel-format conversion
- Externals: `HUD_Class` (base), `current_player_index`, `MotionSensorActive`, `LuaTexturePaletteSize()`, interface color/font accessors

# Source_Files/RenderOther/HUDRenderer_OGL.h
## File Purpose
OpenGL-specific implementation of HUD rendering. Provides concrete drawing operations for the in-game heads-up display by inheriting from `HUD_Class` and overriding abstract rendering methods.

## Core Responsibilities
- Implement motion sensor display updates and rendering
- Draw 2D shapes, text, and textures to the HUD using OpenGL primitives
- Manage entity blips (radar contacts) on the motion sensor
- Handle clipping planes for circular HUD element boundaries (e.g., motion sensor)
- Provide primitive drawing operations (rectangles, filled regions, framed outlines)
- Render message and notification areas

## External Dependencies
- `HUDRenderer.h` ΓÇô base class `HUD_Class`, shared constants (MOTION_SENSOR_SIDE_LENGTH, etc.)
- Game engine types: `shape_descriptor`, `screen_rectangle`, `point2d`, `interface_state_data`
- OpenGL API (actual calls in .cpp implementation, not visible in header)

# Source_Files/RenderOther/HUDRenderer_SW.cpp
## File Purpose
Software-based HUD renderer implementing shape, text, and UI element drawing for the Aleph One game engine. Provides SDL-based implementations of motion sensor updates, texture drawing with rotation/scaling, and basic primitive rendering (rectangles, text).

## Core Responsibilities
- Update and render motion sensor display with dynamic status checks
- Draw shapes to HUD buffer with optional source/destination clipping
- Handle texture drawing with 90┬░ rotation and arbitrary rescaling
- Render text strings with font and color styling
- Draw filled and outlined rectangles for UI elements
- Manage SDL surface conversions for multi-format pixel depth support (8, 16, 32-bit)

## External Dependencies
- **Includes:** `HUDRenderer.h` (base class), `images.h` (SDL and resource loading), `shell.h` (shape/screen functions)
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface`, `SDL_DisplayFormat`, `SDL_BlitSurface`, `SDL_FreeSurface`, `SDL_SetColors`
- **Defined elsewhere:** `_draw_screen_shape[_at_x_y]()`, `_draw_screen_text()`, `_fill_rect()`, `_frame_rect()`, `get_shape_surface()`, `rescale_surface()`, `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `BUILD_DESCRIPTOR()`, `GET_GAME_OPTIONS()`

# Source_Files/RenderOther/HUDRenderer_SW.h
## File Purpose
Defines a software-rendered HUD renderer class (`HUD_SW_Class`) that inherits from the base `HUD_Class`. Provides implementations of abstract rendering methods using CPU-based graphics functions for drawing the player's heads-up display (motion sensor, ammo, energy, oxygen, etc.).

## Core Responsibilities
- Implement motion sensor rendering (update and render logic)
- Provide primitive drawing operations (shapes, text, rectangles, textures) via software rendering
- Override base class abstract methods for entity blip (radar dot) rendering
- Manage clipping regions for motion sensor viewport (though SetClipPlane/DisableClipPlane are no-ops)
- Interface between the high-level HUD update logic and low-level screen drawing functions

## External Dependencies
- **Includes**: `HUDRenderer.h` (base class and HUD types)
- **Base class**: `HUD_Class` (abstract interface with pure virtual drawing/update methods)
- **Types used but not defined here**: `shape_descriptor`, `screen_rectangle`, `point2d`, `short` (likely from cseries.h or platform headers)
- **Implicit dependency**: `screen_drawing.h` functions (called by the `.cpp` implementations to render primitives)
- **Platform conditionals**: Windows-specific `#undef DrawText` to avoid macro conflicts with WinAPI

# Source_Files/RenderOther/images.cpp
## File Purpose
Manages loading and rendering of image resources (PICT pictures and CLUT color tables) from macOS resource forks and WAD files. Handles decompression of PackBits-RLE compressed picture data and provides full-screen picture rendering with optional scrolling.

## Core Responsibilities
- Load PICT, CLUT, sound, and text resources from resource files and WAD archives
- Decompress PackBits RLEΓÇôencoded picture data at depths 1/2/4/8/16/32-bit
- Convert macOS PICT resources to SDL_Surface for rendering
- Render full-screen pictures with keyboard/mouse-controlled scrolling
- Select appropriate picture resource IDs based on current display bit depth
- Convert WAD-format picture/CLUT data to macOS resource format (or vice versa)
- Manage dual image sources (main Images file + per-scenario scenario file)

## External Dependencies
- **SDL/SDL_image:** `SDL_RWops`, `SDL_Surface`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_ReadBE16/32()`, `IMG_LoadTyped_RW()` (JPEG)
- **FileHandler.h:** `OpenedResourceFile`, `OpenedFile`, `LoadedResource`, `FileSpecifier`
- **wad.h:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `free_wad()`
- **shell.h:** `global_idle_proc()`, `alert_user()`
- **screen.h/screen_drawing.h:** `interface_bit_depth` (external), `draw_clip_rect_active`, `draw_clip_rect`
- **byte_swapping.h:** `byte_swap_memory()`
- **cseries.h:** Base types, `FOUR_CHARS_TO_INT()`, `assert()`, `MIN()`
- **MacOS QuickTime** (conditional `#ifdef mac`): `QTNewGWorldFromPtr()`, `OpenPicture()`, `ClosePicture()`, `CopyBits()`, graphics state management

# Source_Files/RenderOther/images.h
## File Purpose

Image and resource management subsystem for the Aleph One game engine. Provides interfaces to load and manipulate game artwork (pictures, sounds, text) from both game resources and scenario files, with dual-platform support for MacOS resource forks and SDL-based systems.

## Core Responsibilities

- Initialize and manage the image/resource loading system
- Load and check existence of picture resources from game files and scenario files
- Calculate and manage color lookup tables (CLUTs) for image rendering
- Draw full-screen pictures and handle picture scrolling with text overlays
- Load sound and text resources from scenario files
- Convert MacOS PICT resources to SDL surfaces
- Perform surface transformations (rescaling, tiling) on SDL graphics objects
- Abstract away platform-specific resource fork handling via FileSpecifier

## External Dependencies

- **FileHandler.h** ΓÇö `LoadedResource`, `FileSpecifier` abstractions for file I/O and resource lifetime.
- **SDL.h** (conditional) ΓÇö `SDL_Surface`, `SDL_RWops` for graphics and file I/O on non-MacOS platforms.
- **Implicit MacOS APIs** ΓÇö On `mac` platform: Carbon/Classic resource manager (`GetCTable`, etc.) via FileHandler.h.

# Source_Files/RenderOther/motion_sensor.cpp
## File Purpose
Implements the motion sensor displayΓÇöa circular HUD radar that tracks nearby monsters and players. Renders entity "blips" with intensity trails, manages entity lifecycle (appear/fade/disappear), and supports both software and OpenGL rendering backends with customizable monster type classifications.

## Core Responsibilities
- Initialize and reset motion sensor display state per level/player
- Scan world for monsters/players within sensor range using distance and visibility checks
- Track entity positions across multiple frames, managing fade-out animations for departing entities
- Render entity blips using bitmap operations with circular clipping region
- Handle network compass display (multiplayer position indicators)
- Provide XML-based customization of monster type display classes and sensor parameters
- Support both software (pixel-based) and OpenGL rendering implementations

## External Dependencies
- **Notable includes:** `monsters.h` (monster/player data), `map.h` (world geometry), `render.h` (view/rendering), `interface.h` (shapes, bitmaps), `player.h` (player data), `network_games.h` (compass state), `HUDRenderer_SW.h` / `HUDRenderer_OGL.h` (renderer-specific drawing).
- **Key external functions/symbols:** `get_object_data()`, `get_monster_data()`, `get_player_data()`, `guess_distance2d()`, `transform_point2d()`, `get_shape_bitmap_and_shading_table()`, `DrawShapeAtXY()` (OGL), `get_interface_rectangle()`, `get_network_compass_state()` (defined elsewhere).
- **Game world globals:** `monsters[]`, `dynamic_world->tick_count`, `static_world->environment_flags` (magnetic environment check).

# Source_Files/RenderOther/motion_sensor.h
## File Purpose
Header file declaring the motion sensor system interface for the Aleph One game engine. Provides functions to initialize, manage, and query the in-game motion tracker that displays enemy/object detection, plus XML configuration support.

## Core Responsibilities
- Initialize motion sensor with shape descriptors for mounts, aliens, friendly units, enemies, and compass
- Reset motion sensor state per monster/entity
- Query motion sensor state changes (for dirty-flag/update optimization)
- Dynamically adjust motion sensor detection range
- Provide XML element parser for motion sensor configuration

## External Dependencies
- **XML_ElementParser.h** ΓÇô Base class for XML parsing framework.
- **shape_descriptor** ΓÇô Type defined elsewhere; references shape data for rendering motion sensor icons.
- Implementation in `motion_sensor.c` (referenced in comment).

# Source_Files/RenderOther/OGL_Blitter.cpp
## File Purpose
Implements an OpenGL texture-based blitter that converts SDL surfaces into tiled GPU textures for efficient rendering. Manages the transformation of 2D SDL surface data into OpenGL texture tiles (256├ù256), with matrix setup and render operations.

## Core Responsibilities
- Convert SDL surfaces into OpenGL textures, tiling large images into 256├ù256 chunks
- Calculate and manage scaling factors between source surface and destination screen coordinates
- Set up orthographic projection matrices for 2D rendering
- Render tiled textures as quads with proper texture coordinates and scaling
- Manage GPU texture lifecycle (creation, upload, deletion)
- Preserve and restore OpenGL state during rendering operations

## External Dependencies
- **SDL (Simple DirectMedia Layer):** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`
- **OpenGL:** `GL/gl.h`, `GL/glu.h` (platform-conditional includes); `glGenTextures()`, `glBindTexture()`, `glTexImage2D()`, matrix/projection functions, immediate-mode rendering
- **Platform conditionals:** Handles macOS (`__APPLE__`/`__MACH__`), Windows (`__WIN32__`), and Linux endianness (`ALEPHONE_LITTLE_ENDIAN`)
- **Included header:** `OGL_Blitter.h` (class definition, static tile_size)

# Source_Files/RenderOther/OGL_Blitter.h
## File Purpose
OpenGL utility class for blitting (rendering) SDL surfaces to the framebuffer. Optimizes large images by tiling them into 256├ù256 chunks and managing OpenGL texture state during rendering.

## Core Responsibilities
- Encapsulate OpenGL matrix setup/teardown for blitting operations
- Tile SDL surfaces into manageable texture sizes (256├ù256)
- Track texture references and blit rectangles for multi-tile draws
- Handle platform-specific OpenGL header inclusion (macOS, Linux, Windows)
- Coordinate SDL surface data with OpenGL rendering pipeline

## External Dependencies
- **OpenGL:** `<OpenGL/gl.h>`, `<OpenGL/glu.h>` (macOS); `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glext.h>` (Linux/Windows)
- **SDL:** `<SDL/SDL.h>` (surface and rect types)
- **Engine:** `cseries.h` (common type definitions and macros)
- **Standard Library:** `<vector>` (tile storage)

# Source_Files/RenderOther/OGL_LoadScreen.cpp
## File Purpose
Implements OpenGL-based loading screens for the Aleph One game engine. Handles image file loading, aspect-ratio-aware scaling, and rendering with an optional progress bar overlay during asset loading.

## Core Responsibilities
- Singleton instance management via lazy initialization
- Image file loading and SDL surface conversion with endianness handling
- Aspect-ratio-aware or stretch-fill image scaling and centering
- Progress bar rendering (vertical or horizontal) with configurable dimensions and colors
- OpenGL blitter initialization, setup, and cleanup
- Resource deallocation and state reset

## External Dependencies
- **Headers**: OGL_LoadScreen.h, screen.h, OGL_Blitter.h, ImageLoader.h, cseries.h
- **OpenGL**: gl.h / GL/gl.h ΓÇö glBindTexture, glColor3us, glBegin, glVertex3f, glEnd
- **SDL**: SDL_CreateRGBSurfaceFrom, SDL_Rect, SDL_Surface, SDL_GetVideoSurface
- **Game engine**: OGL_ClearScreen(), OGL_SwapBuffers(), bound_screen() (defined elsewhere); FileSpecifier, ImageDescriptor, OGL_Blitter classes
- **Platform**: WindowPtr, GetWindowPort, GetPortBounds (Mac OS Classic only, guarded by #if defined(mac))

# Source_Files/RenderOther/OGL_LoadScreen.h
## File Purpose
Singleton class managing OpenGL-rendered load screens displayed during game level loading. Displays a background image with optional progress indicator percentage. Coordinates image loading, texture management, and on-screen rendering via OpenGL blitting.

## Core Responsibilities
- Load and cache image files for display during level transitions
- Render background image via OpenGL with optional stretching/positioning
- Display and update progress percentage indicator during loading
- Manage OpenGL texture references and lifecycle
- Provide start/stop lifecycle controls for load screen visibility
- Store and expose UI color palette for progress indicator rendering

## External Dependencies
- **OpenGL:** Platform-specific headers (`OpenGL/gl.h`, `GL/gl.h`, etc.)
- **OGL_Blitter** (Source_Files/RenderOther/OGL_Blitter.h) ΓÇô textured quad rendering
- **ImageLoader** (Source_Files/RenderMain/ImageLoader.h) ΓÇô image file loading and `ImageDescriptor`
- **cseries.h** ΓÇô cross-platform types (`vector`, SDL, `rgb_color`)
- **SDL** ΓÇô low-level graphics/windowing (transitively via cseries.h)

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

# Source_Files/RenderOther/overhead_map.h
## File Purpose
Defines the interface for rendering overhead maps in multiple modes (saved game preview, checkpoint, game map). Declares the data structure for overhead map configuration and provides access to XML configuration parsing for this subsystem.

## Core Responsibilities
- Define overhead map rendering modes and scale constraints
- Declare the configuration data structure for overhead map rendering
- Provide the main rendering entry point
- Expose XML parser for loading overhead map settings from configuration

## External Dependencies
- `XML_ElementParser.h` ΓÇô base class for XML element parsing; provides hierarchical configuration parsing infrastructure
- No standard library includes visible; likely uses game engine type definitions (`world_point2d`, `short`) from elsewhere

# Source_Files/RenderOther/OverheadMap_OGL.cpp
## File Purpose
OpenGL implementation of an overhead map renderer for the Marathon/Aleph One game engine. Subclass of `OverheadMapClass` that renders the in-game map efficiently using vertex caching and batch drawing to avoid CPU bottlenecks at high resolutions.

## Core Responsibilities
- Initialize and configure OpenGL state for 2D orthographic map rendering
- Batch and cache polygon geometry for efficient multi-draw rendering
- Batch and cache line segments with dynamic color and width changes
- Render game entities (players, objects, monsters) as geometric primitives
- Render text annotations on the map with layout control
- Accumulate and render movement paths as line strips
- Manage color and pen-size state to minimize redundant state changes

## External Dependencies
- **OpenGL headers:** `<GL/gl.h>`, `<GL/glu.h>` (platform-conditional includes for macOS, Windows, Linux).
- **AGL (macOS only):** `<AGL/agl.h>` for font context.
- **cseries.h:** Core utility macros and types.
- **OverheadMap_OGL.h:** Class definition and member declarations.
- **map.h:** World geometry structures (`world_point2d`, `world_point3d`, `rgb_color`, `angle`), constants (`FULL_CIRCLE`), and global vertex/endpoint arrays (via `GetVertex()`, `GetVertexStride()`, `GetFirstVertex()` macros/functions not defined in this file).

**Defined elsewhere (inferred):**
- `SetColor()`, `ColorsEqual()`: Utility inlines in this file.
- `GetVertex()`, `GetVertexStride()`, `GetFirstVertex()`: Likely defined in parent class or utility header.

# Source_Files/RenderOther/OverheadMap_OGL.h
## File Purpose
Defines `OverheadMap_OGL_Class`, an OpenGL-specific subclass of `OverheadMapClass` for rendering the overhead/automap in-game HUD. Provides graphics-API-specific implementations of map rendering operations including polygons, lines, entities, and text with geometry caching for batched drawing.

## Core Responsibilities
- Override virtual rendering methods from `OverheadMapClass` with OpenGL implementations
- Manage rendering lifecycle through begin/end pairs for polygons, lines, and the overall render frame
- Cache polygon and line geometry for batched rendering
- Draw map primitives: polygons (terrain), lines (walls/elevation), things (items/monsters), player indicator
- Support text rendering with font and justification settings
- Buffer path points for visualizing entity/monster movement trails

## External Dependencies
- `#include <vector>` ΓÇô STL for geometry caches
- `#include "OverheadMapRenderer.h"` ΓÇô parent class `OverheadMapClass`, types (`rgb_color`, `world_point2d`, `FontSpecifier`, `angle`)
- External symbols: `rgb_color`, `world_point2d`, `angle`, `FontSpecifier` (defined elsewhere in engine)

# Source_Files/RenderOther/OverheadMap_SDL.cpp
## File Purpose
SDL-based implementation of the overhead map renderer for the Aleph One game engine. Provides concrete drawing implementations for map visualization, rendering polygons, lines, objects, players, text, and movement paths to an SDL surface.

## Core Responsibilities
- Render filled polygons (map regions) with color
- Render lines (walls, grid) with variable pen width
- Render map objects (scenery, items) as rectangles or octagonal circles
- Render player position and facing direction as a triangle
- Render text annotations on the map
- Maintain and render player movement path traces

## External Dependencies
- **SDL library** (`SDL.h` via cseries.h): surface operations, color mapping, rectangle filling
- **Parent class** `OverheadMapClass` (from OverheadMapRenderer.h): defines interface; `GetVertex()` accessor
- **Map module** (`map.h`): geometry types (`world_point2d`, `world_distance`, `angle`)
- **Screen drawing** (`screen_drawing.h`): low-level SDL drawing functions (`draw_polygon`, `draw_line`, `draw_text`); font handling
- **Global state** (`screen_sdl.cpp`): `draw_surface` SDL_Surface pointer

# Source_Files/RenderOther/OverheadMap_SDL.h
## File Purpose
SDL-specific implementation of the overhead map renderer. This class is a concrete subclass of `OverheadMapClass` that implements all virtual rendering methods using SDL graphics primitives for drawing the in-game mini-map.

## Core Responsibilities
- Override base class virtual methods to render map elements using SDL
- Draw polygons (terrain/level geometry) with colors
- Draw lines (walls/elevation changes) with variable pen sizes
- Draw things (entities: monsters, items, projectiles) as shapes
- Draw the player indicator with directional orientation
- Render text annotations for map labels
- Manage path visualization for checkpoint navigation

## External Dependencies
- `OverheadMapRenderer.h` ΓÇö base class definition; includes game types (`rgb_color`, `world_point2d`, `angle`), font abstraction (`FontSpecifier`), and configuration data structures
- SDL library ΓÇö graphics rendering backend (not shown in header; implementation in `.cpp`)
- Standard game types ΓÇö `world_point2d`, `angle`, `rgb_color` (defined elsewhere)

# Source_Files/RenderOther/OverheadMapRenderer.cpp
## File Purpose
Implements the overhead map renderer for the Aleph One game engine. Transforms world geometry and objects into screen-space for the automap display, handling viewport-aligned rendering of polygons, lines, entities, annotations, and paths with color coding for different terrain and object types.

## Core Responsibilities
- Render visible polygons with terrain-type coloring (water, lava, platforms, hazard zones)
- Draw map lines with elevation and obstruction indicators
- Transform world coordinates to screen-space with viewport culling
- Render game objects (players, monsters, items, projectiles) with visibility filtering
- Display map annotations and the level name on the overhead view
- Generate and restore "false automaps" for checkpoint map rendering (flooded visibility)
- Manage path visualization for waypoint routes
- Respect game options for visibility of aliens, items, and projectiles

## External Dependencies
- **Notable includes:** `cseries.h` (cross-platform compatibility), `OverheadMapRenderer.h` (class definition), `flood_map.h` (pathfinding), `media.h`, `platforms.h`, `player.h`, `render.h` (world rendering)
- **External symbols (defined elsewhere):**
  - World/game state: `dynamic_world`, `static_world`, `objects`, `saved_objects`, `local_player`
  - Data accessors: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_platform_data()`, `get_monster_data()`, `get_player_data()`
  - Automap state: `automap_lines`, `automap_polygons`
  - Pathfinding: `flood_map()`, `path_peek()`, `GetNumberOfPaths()`
  - Macros: `POLYGON_IS_IN_AUTOMAP()`, `TEST_STATE_FLAG()`, `SET_STATE_FLAG()`, `LINE_IS_IN_AUTOMAP()`, `LINE_IS_SOLID()`, `LINE_IS_VARIABLE_ELEVATION()`, `LINE_IS_LANDSCAPED()`, `WORLD_TO_SCREEN()`, `PLATFORM_IS_SECRET()`, `SLOT_IS_USED()`, etc.
  - Rendering hooks (virtual methods in subclass)

# Source_Files/RenderOther/OverheadMapRenderer.h
## File Purpose
Defines the base class and configuration structures for rendering the overhead (top-down) map in the Aleph One game engine. Provides virtual render interface for subclasses to implement graphics-API-specific drawing (e.g., OpenGL, software rasterization).

## Core Responsibilities
- Define configuration data for overhead map appearance (colors, fonts, shapes, scales)
- Declare virtual interface for rendering polygons, lines, entities, annotations, and paths
- Manage coordinate transformation for viewport-relative map rendering
- Support automap generation including "false" automaps for checkpoint-based discovery
- Provide template-method orchestration of rendering passes (begin/draw/end stages)

## External Dependencies
- **world.h**: `world_point2d`, `angle` type definitions and vector types
- **map.h**: `endpoint_data`, `get_endpoint_data()`, map geometry accessors
- **monsters.h**: `NUMBER_OF_MONSTER_TYPES` constant
- **overhead_map.h**: `overhead_map_data` structure, scale constants
- **shape_descriptors.h**: `shape_descriptor` type
- **FontHandler.h**: `FontSpecifier` class for font management
- **shell.h**: `_get_player_color()` function and RGBColor type
- **cseries.h**: Standard types, macros, RGB color definitions

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

## External Dependencies
- **color_table** (defined elsewhere) ΓÇô Palette data structure
- **screen_mode_data** (SHELL.H / PREFERENCES.H) ΓÇô Mode configuration
- **Rect** (QuickDraw) ΓÇô Rectangle bounds
- **OpenGL** (implicit) ΓÇô Hardware acceleration backend option
- **Pfhortran** (noted in comments) ΓÇô Script engine for HUD element control

# Source_Files/RenderOther/screen_definitions.h
## File Purpose
Defines base resource IDs for all screen types in the rendering system. These constants serve as anchors for a three-tiered numbering scheme: 8-bit variants use the base ID, 16-bit variants add +10,000, and 32-bit variants add +20,000.

## Core Responsibilities
- Provide compile-time constants for screen resource identification
- Establish a consistent offset pattern (100-point gaps) for different screen categories
- Support multi-bit-depth asset management without hardcoding variant IDs

## External Dependencies
None beyond C standard library (enum syntax only).

# Source_Files/RenderOther/screen_drawing.cpp
## File Purpose
Manages 2D UI and HUD rendering by drawing shapes, text, rectangles, and polygons to SDL surfaces. Provides a portable interface for screen output redirection and maintains interface resources (colors, fonts, rectangles).

## Core Responsibilities
- Initialize interface resources (fonts, colors, rectangles) from data and XML
- Redirect drawing output between screen, world, HUD, and terminal buffers
- Render shapes/sprites with SDL blitting and 8-bit surface handling
- Render text with bitmap and TrueType fonts, supporting wrapping and positioning
- Draw primitives (lines, rectangles, polygons) with Cohen-Sutherland and Sutherland-Hodgman clipping
- Manage global clipping rectangle state for all drawing operations
- Parse XML configuration for interface geometry and resources

## External Dependencies
- **SDL**: `SDL.h`, `SDL_ttf.h` (TrueType rendering, conditional)
- **Engine core**: cseries.h, map.h, interface.h, shell.h, screen.h, fades.h
- **XML parsing**: XML_ElementParser.h, ColorParser.h, FontHandler.h
- **Font/glyph rendering**: sdl_fonts.h (sdl_font_info, font_info, TextSpec)
- **Shapes**: shape_descriptors.h, `get_shape_surface()` (defined elsewhere)

**External symbols**: `world_pixels`, `HUD_Buffer`, `Term_Buffer` (screen_sdl.cpp); `environment_preferences` (prefs); font/color style constants (styleBold, styleItalic, etc.); SDL and TTF library functions; `_get_interface_color()` overloads in shell.h.

# Source_Files/RenderOther/screen_drawing.h
## File Purpose
Interface header for screen drawing and rendering operations in the Aleph One game engine. Defines UI layout (rectangles, colors, fonts) and provides functions for drawing shapes, text, and manipulating the screen framebuffer. Supports both classic and SDL-based rendering backends with XML configuration for interface elements.

## Core Responsibilities
- Define rectangle IDs for game HUD and interface buttons (player name, weapon display, menu buttons, etc.)
- Define color palette indices for UI rendering (weapons, inventory, computer interface colors)
- Define font identifiers for different UI text types (interface, weapon names, computer terminal, net stats)
- Provide shape/sprite rendering primitives with source/destination clipping
- Provide text rendering with measurement and justification support
- Manage rendering "ports" (current render target context)
- Support HUD buffering and screen manipulation (erase, fill, scroll, frame)
- Enable XML-driven UI layout configuration

## External Dependencies
- **XML_ElementParser.h**: XML parsing for UI layout configuration
- **shape_descriptors.h**: Shape/sprite ID encoding (collection + shape bits)
- **sdl_fonts.h**: Font abstraction (font_info, sdl_font_info, ttf_font_info classes)
- **SDL** (conditional): Graphics library for surface operations, geometry drawing
- Standard C: stdio, strlen, sprintf (via indirect includes)
- Defined elsewhere: `FontSpecifier` class, `rgb_color` type, underlying rendering backend

# Source_Files/RenderOther/screen_sdl.cpp
## File Purpose
SDL-based screen management and rendering system for the Aleph One game engine. Handles display initialization, mode switching (resolution/fullscreen/acceleration), rendering surface allocation, and frame composition (world view, HUD, terminal, overhead map). Integrates with OpenGL for accelerated rendering when available.

## Core Responsibilities
- Initialize and tear down SDL display surfaces and rendering contexts
- Manage screen mode changes (resolution, fullscreen toggling, bit depth, acceleration)
- Allocate and reallocate off-screen rendering buffers (world_pixels, HUD_Buffer, Term_Buffer)
- Route frame rendering to software or OpenGL acceleration paths
- Composite game world, HUD, terminal, and overlay elements to display
- Handle color palette management and gamma correction
- Manage OpenGL initialization, viewport setup, and context lifecycle
- Scale low-resolution buffers 2x for display when hardware acceleration is disabled

## External Dependencies
**Includes (notable):**
- `SDL.h`, `SDL_opengl.h` ΓÇö SDL and OpenGL APIs
- `world.h`, `map.h`, `render.h` ΓÇö Game world, map, rendering definitions
- `shell.h`, `interface.h`, `player.h` ΓÇö Core game state (current_player, game state)
- `OGL_Blitter.h`, `OGL_Render.h` ΓÇö OpenGL rendering backend
- `ViewControl.h` ΓÇö Camera/FOV utilities
- `screen_drawing.h` ΓÇö Drawing primitives
- `Crosshairs.h`, `overhead_map.h`, `computer_interface.h` ΓÇö UI/overlay rendering

**Defined elsewhere (key symbols used):**
- `current_player`, `dynamic_world` ΓÇö Global game state (defined in world/player modules)
- `world_view` ΓÇö View/camera structure (allocated here, manipulated in render.cpp)
- `render_view()` ΓÇö Main 3D rendering function (defined in render.cpp)
- `screen_mode` ΓÇö Global screen_mode_data structure (defined in shell/preferences)
- `ViewSizes[]` ΓÇö Array of layout configs (defined in game_window.c)
- `OGL_StartRun()`, `OGL_StopRun()`, `OGL_RenderCrosshairs()`, `OGL_DrawHUD()`, `OGL_SetWindow()`, `OGL_SwapBuffers()` ΓÇö OpenGL lifecycle & rendering (defined in OGL_Render.cpp)
- `set_overhead_map_status()`, `set_terminal_status()` ΓÇö Overlay state management
- `update_fps_display()`, `DisplayPosition()`, `DisplayNetMicStatus()`, `DisplayMessages()`, `DisplayInputLine()` ΓÇö HUD text rendering

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

## External Dependencies
- **Includes:** `<stdarg.h>` (variadic), `snprintf.h` (safe formatting), `Console.h` (debug console), `screen_drawing.h` (drawing primitives)
- **Symbols defined elsewhere:** `world_view`, `current_player`, `current_player_index`, `dynamic_world`, `OGL_MapActive`, `OGL_IsActive()`, `OGL_RenderText()`, `gamma_correct_color_table()`, `change_screen_mode()`, `start_render_effect()`, `View_DoFoldEffect()`, `dirty_terminal_view()`, `player_in_terminal_mode()`, `get_player_data()`, `current_netgame_allows_microphone()`, `NetGetLatency()`, `GET_GAME_OPTIONS()`, `_get_interface_color()`, `GetOnScreenFont()`, `start_tunnel_vision_effect()`, SDL functions, OpenGL functions

# Source_Files/RenderOther/sdl_fonts.cpp
## File Purpose
Manages font loading, caching, and text metrics for the Aleph One game engine. Supports both legacy bitmap fonts (from Mac resources) and modern TrueType fonts with style variants (bold, italic, etc.).

## Core Responsibilities
- Initialize font system by scanning data directories for font resources
- Load and cache bitmap fonts from FOND/NFNT/FONT resource structures
- Load and cache TrueType fonts with automatic fallback chains for style variants
- Convert bitmap font data from 1-bit-per-pixel to 1-byte-per-pixel pixmaps
- Calculate text width for both font types, handling style codes and shadows
- Truncate text to fit within max width
- Parse and apply inline style codes (|b, |i, |p) in text strings
- Manage reference counting for cached fonts

## External Dependencies
- **Resources:** `resource_manager.h`, `byte_swapping.h` (Mac resource format, big-endian)
- **File I/O:** `FileHandler.h` (FileSpecifier), `Logging.h`
- **SDL:** `<SDL_endian.h>`, optionally SDL_ttf (TTF_Font, TTF_*) 
- **STL:** `<vector>`, `<map>`, `<string>`
- **Boost:** `<boost/tokenizer.hpp>` (text parsing)
- **External symbols:** `data_search_path`, `fix_missing_overhead_map_fonts()`, `fix_missing_interface_fonts()`, `mac_roman_to_unicode()`, `environment_preferences`, `_draw_text()` (in screen_drawing.cpp)

# Source_Files/RenderOther/sdl_fonts.h
## File Purpose
Defines font rendering abstractions for the Aleph One engine, supporting both bitmap-based and TrueType fonts via SDL. Provides unified interface for text measurement, drawing, truncation, and styled text handling.

## Core Responsibilities
- Abstract interface for font operations (metrics, text rendering, width calculation)
- Bitmap font implementation using pre-loaded pixmaps and kerning tables
- TrueType font support (when SDL_TTF enabled) with multiple style variants (bold, italic, underline)
- Text layout utilities: styled text handling, width measurement, text truncation
- Font lifecycle management (load/unload with reference counting)
- UTF-8 and MacRoman text encoding support

## External Dependencies
- **FileHandler.h**: `LoadedResource` class (manages resource lifetime for bitmap font pixmap)
- **SDL_ttf.h**: Conditional, provides `TTF_Font` type and metrics/rendering functions (when `HAVE_SDL_TTF` defined)
- **boost/tuple/tuple.hpp**: Conditional, defines `ttf_font_key_t` for font caching
- **Standard library**: `<string>`, `<cstddef>` (implicit from includes)
- **tags.h** (via FileHandler.h): Typecode definitions

# Source_Files/RenderOther/TextLayoutHelper.cpp
## File Purpose
Implements a layout helper that manages placement of non-overlapping rectangles (likely for UI/HUD text rendering in Marathon: Aleph One). Given a new rectangle's horizontal extent and minimum vertical position, it calculates a safe vertical position that avoids collisions with previously reserved rectangles.

## Core Responsibilities
- Track active rectangle reservations via horizontal/vertical boundary events
- Insert new reservations into a sorted list of horizontal boundaries
- Detect overlapping rectangles by horizontal extent
- Resolve vertical collisions by adjusting placement upward iteratively
- Manage memory cleanup for all tracked reservations

## External Dependencies
- `<vector>` ΓÇö `CollectionOfReservationEnds` storage
- `<set>` ΓÇö `std::multiset<Reservation*>` for tracking overlaps
- `<assert.h>` ΓÇö runtime bounds/sanity checks
- `TextLayoutHelper.h` ΓÇö type definitions (`Reservation`, `ReservationEnd`)

# Source_Files/RenderOther/TextLayoutHelper.h
## File Purpose
Utility class for managing non-overlapping rectangular space allocations. Provides a simple reservation system to place text or UI elements without overlap, tracking used regions and computing safe placement coordinates.

## Core Responsibilities
- Reserve rectangular space with guaranteed non-overlap detection
- Track horizontal and vertical boundaries of all reservations
- Compute minimum Y-coordinate that avoids existing reservations
- Clear and reset all active reservations

## External Dependencies
- `<vector>` (STL container for tracking reservation boundaries)
- `using namespace std;`

# Source_Files/RenderOther/TextStrings.cpp
## File Purpose
Implements a text string management system that replaces MacOS STR# resources. Provides storage and retrieval of string collections indexed by resource ID, supporting both Pascal and C-string formats via XML configuration. Part of the Aleph One game engine.

## Core Responsibilities
- Manage hierarchical string collections (StringSet) organized by resource ID
- Store strings internally as Pascal strings (length byte + data + null terminator) for dual format support
- Provide public API for inserting, retrieving, counting, and deleting strings
- Parse string definitions from XML markup (`<stringset>` and `<string>` elements)
- Dynamically grow internal string arrays when needed
- Maintain linked list of all active string sets in memory

## External Dependencies

- `<string.h>` ΓÇö `memcpy()`, `strlen()`
- `"cseries.h"` ΓÇö `objlist_clear()`, `objlist_copy()`, cross-platform macros
- `"XML_ElementParser.h"` ΓÇö Base class `XML_ElementParser`, helper functions `StringsEqual()`, `ReadInt16Value()`, `DeUTF8_Pas()`
- Assumed elsewhere: `Str255` typedef (MacOS Pascal string type, `unsigned char[256]`)

# Source_Files/RenderOther/TextStrings.h
## File Purpose
Header file declaring a text-string repository interface for the Aleph One game engine. Replaces macOS STR# resource management with a unified API for storing, retrieving, and managing text strings organized by resource ID and index. Supports both Pascal-format (length-prefixed) and C-style (null-terminated) strings.

## Core Responsibilities
- Store and retrieve strings by resource ID and index
- Convert between Pascal-format and C-style string representations
- Query string set presence and string count
- Delete individual strings or entire string sets
- Provide XML parsing interface for string configuration
- Manage in-memory string repository

## External Dependencies
- `XML_ElementParser` ΓÇô forward-declared; defined elsewhere; used for XML string configuration

# Source_Files/RenderOther/ViewControl.cpp
## File Purpose
Implements view controller for the Aleph One game engine, managing camera field-of-view (FOV), display effects (fold, static, teleport), landscape texture rendering options, and on-screen font settings. Includes XML parsing for game-developer-friendly configuration of these parameters.

## Core Responsibilities
- Manage view effect toggles (overhead map, fold/static effects, teleport effects)
- Control and interpolate field-of-view (FOV) between normal, tunnel vision, and extra-vision states
- Store and retrieve landscape texture configuration per collection/frame
- Parse and validate XML configuration for view and landscape settings
- Provide on-screen font initialization and access

## External Dependencies
- **Standard library:** `<vector>`, `<string.h>`
- **Engine core:** `cseries.h` (macros, type defs), `world.h` (angle, world_distance types)
- **Rendering:** `ViewControl.h` (declarations); `FontHandler.h` (FontSpecifier)
- **XML parsing:** `XML_ElementParser` (base class; defined elsewhere)
- **External symbols used:**
  - `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()` macros (shape_descriptors.h)
  - `StringsEqual()`, `ReadBoundedNumericalValue()`, `ReadBooleanValueAsBool()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadInt16Value()`, `UnrecognizedTag()`, `AttribsMissing()` (XML parsing utilities, defined elsewhere)
  - `FontSpecifier::Init()`, `Font_SetArray()`, `Font_GetParser()` (font management, defined elsewhere)

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

## External Dependencies

- **world.h:** Provides `angle` type (int16 azimuth angles in 0ΓÇô511 range for a full circle).
- **FontHandler.h:** Provides `FontSpecifier` class for on-screen text rendering.
- **shape_descriptors.h:** Provides `shape_descriptor` type (uint16 encoding collection, shape, and CLUT).
- **XML_ElementParser.h:** Provides `XML_ElementParser` base class for declarative XML-driven configuration.


