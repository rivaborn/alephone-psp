# Subsystem Overview

## Purpose

Source_Files/RenderOther is the rendering and user interface subsystem for Aleph One, managing on-screen display of the HUD, menus, effects, text, maps, and camera systems. It bridges the game world (3D geometry, player state, entities) with platform-specific rendering backends (OpenGL, SDL software rasterization) and provides cross-platform abstractions for fonts, images, and drawable primitives.

## Key Files

| File | Role |
|------|------|
| ChaseCam.cpp/h | Third-person follow camera with spring physics and collision detection |
| computer_interface.cpp/h | In-game terminal/computer display system with text rendering and input handling |
| fades.cpp/h | Screen fade and color tint effects with gamma correction |
| FontHandler.cpp/h | Font specification, loading, and rendering (bitmap and OpenGL paths) |
| game_window.cpp/h | Game window initialization and HUD buffer management |
| HUDRenderer.cpp/h | Base class for HUD rendering with dirty-flag optimization |
| HUDRenderer_OGL.cpp/h | OpenGL-accelerated HUD renderer implementation |
| HUDRenderer_SW.cpp/h | Software (SDL-based) HUD renderer implementation |
| images.cpp/h | Image and sound resource loading (PICT, CLUT, WAD extraction, RLE decompression) |
| motion_sensor.cpp/h | Circular radar display tracking nearby monsters and players |
| OGL_Blitter.cpp/h | OpenGL texture tiling and rendering utility for large images |
| OGL_LoadScreen.cpp/h | OpenGL-rendered loading screen with progress indicator |
| overhead_map.cpp/h | Overhead/automap render dispatcher selecting backend (SDL or OpenGL) |
| OverheadMap_OGL.cpp/h | OpenGL-accelerated overhead map renderer with geometry caching |
| OverheadMap_SDL.cpp/h | SDL-based overhead map renderer |
| OverheadMapRenderer.cpp/h | Base class orchestrating map rendering (polygons, lines, entities, text) |
| screen.h | Screen mode, display, color table, and HUD lifecycle declarations |
| screen_definitions.h | Resource ID constants for multi-bit-depth screen assets |
| screen_drawing.cpp/h | 2D primitives (shapes, text, rectangles, polygons) with clipping |
| screen_sdl.cpp | SDL display initialization, mode switching, buffer allocation, frame composition |
| screen_shared.h | Shared screen state (view presets, message buffering, debug display) |
| sdl_fonts.cpp/h | Bitmap and TrueType font loading, caching, and text metrics |
| TextLayoutHelper.cpp/h | Utility for non-overlapping rectangular space reservation |
| TextStrings.cpp/h | String repository replacing macOS STR# resources |
| ViewControl.cpp/h | Field-of-view, view effects (fold, static, teleport), landscape texture configuration |

## Core Responsibilities

- **Chase camera**: Maintain follow-camera state with spring/damping physics, raycast collisions against geometry
- **Computer terminals**: Render formatted text with styling, handle player navigation input, manage terminal state lifecycle and serialization
- **Screen effects**: Apply cinematic fades, color tints, flash effects, environmental overlays (water/lava); blend priorities and perform gamma correction
- **Font system**: Load platform-specific fonts (bitmap/TrueType), calculate metrics, render text with style codes and wrapping
- **Game window/HUD**: Allocate and manage rendering buffers; coordinate per-frame HUD updates using dirty-flag pattern
- **HUD rendering**: Abstract platform-specific rendering (shapes, text, textures, rectangles, motion sensor, network stats)
- **Hardware HUD**: Implement OpenGL-accelerated HUD with texture atlasing and display lists
- **Software HUD**: Implement SDL-based CPU rasterization for HUD elements
- **Image resources**: Load PICT pictures and CLUT color tables from resource forks and WAD files with RLE decompression
- **Motion sensor**: Track and render nearby entities with distance/visibility checks and fade animations
- **Overhead maps**: Render terrain, walls, entities, annotations, and player paths with world-to-screen transformation and viewport culling
- **Screen management**: Initialize SDL display, switch modes (resolution, fullscreen, bit depth), allocate off-screen buffers, composite frame layers
- **2D drawing**: Provide shape/text/polygon rendering with Cohen-Sutherland and Sutherland-Hodgman clipping
- **View controller**: Manage field-of-view transitions (normal/tunnel/extravision), toggle visual effects, configure landscape textures
- **Text layout**: Reserve and track non-overlapping rectangular regions for UI element placement
- **String management**: Store and retrieve text strings indexed by resource ID (replacing macOS STR# mechanism)

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- Chase camera state query and activation control
- Screen rendering and mode switching API
- HUD element marking (dirty flags for weapon, ammo, shield, oxygen, inventory)
- Computer terminal enter/exit and input processing
- Screen fade and visual effect triggering
- Font and text rendering for UI and HUD
- Text layout reservation and placement
- Overhead map display control
- Field-of-view and view effect management

**Consumes from other subsystems:**
- **Map/World** (map.h): Polygon/line/endpoint geometry, get_polygon_data(), get_line_data(), endpoint queries for collision and rendering
- **Player** (player.h): Current player position, orientation, equipment, health, oxygen, inventory state
- **Monsters/Entities** (monsters.h): Entity positions, types, visibility for motion sensor and overhead map
- **Configuration** (preferences.h): Chase camera parameters, fade settings, font specifications, HUD layout preferences
- **Game State** (dynamic_world, static_world globals): Tick count, environment flags, automap arrays, game mode
- **Network** (network.h): Multiplayer restrictions, latency, compass state, microphone status
- **Audio** (SoundManager.h): Sound playback for terminal logon, HUD interactions
- **Scripting** (lua_script.h): Lua callbacks for terminal exit, HUD element control
- **XML Configuration**: Fade parameters, font definitions, interface layout, motion sensor settings, overhead map appearance

## Runtime Role

**Initialization:**
- `initialize_motion_sensor()` and font system setup called at engine startup
- SDL rendering context created, HUD buffers allocated, resource fonts loaded

**Per-frame (game loop):**
- `update_everything()` called each frame to update HUD dirty elements (weapons, ammo, shield, oxygen, inventory)
- Chase camera position updated once per tick
- Screen fades advanced based on elapsed ticks
- View effects (FOV, fold transitions) interpolated toward target values

**Rendering (composition):**
- Core rendering loop routes to `screen_sdl.cpp:render_screen()`
- Renders world view via hardware (OpenGL) or software rasterization path
- Composites game world, HUD layer, overhead map, terminal interface, crosshairs into final framebuffer
- Swaps buffers or updates SDL display surface

**Input handling:**
- Terminal mode captures player navigation/abort input
- Inventory scrolling input processed
- Chase camera can be toggled on/off

**Shutdown:**
- Rendering resources deallocated, buffers freed, OpenGL context destroyed

## Notable Implementation Details

- **Multi-backend architecture**: Abstract HUD_Class base class with OpenGL (HUDRenderer_OGL) and SDL (HUDRenderer_SW) subclasses allows runtime selection based on hardware acceleration availability
- **Dirty-flag optimization**: HUD updates only redraw modified elements (weapon, ammo, shield, oxygen) each frame to conserve CPU/GPU cycles on resource-constrained PSP hardware
- **Font flexibility**: Supports both legacy bitmap fonts loaded from macOS FOND/NFNT resources and modern TrueType fonts with automatic fallback chains (bold, italic, underline variants)
- **Resource extraction**: PICT and CLUT resources loaded from both macOS resource forks and WAD archives; PackBits RLE decompression handles multiple pixel depths (1/2/4/8/16/32-bit)
- **Chase camera physics**: Implements spring-mass-damper system with inertia history buffer and geometry raycasting to prevent camera clipping through walls
- **Motion sensor optimization**: Entity position tracking with frame-to-frame fading; visibility computed per entity to reduce rendering load
- **Overhead map caching**: Polygon and line geometry batched and cached for efficient multi-draw rendering, minimizing CPU-GPU synchronization
- **XML-driven configuration**: Fade effects, HUD layout, font specifications, interface colors, and motion sensor parameters all declaratively configurable via XML parsing framework
- **Platform conditionals**: Rendering path selection (OpenGL vs SDL) and pixel format conversions (ARGB 1555/8888) conditioned on compile-time platform detection and runtime hardware capabilities
- **Screen composition**: Multiple layers (world_pixels, HUD_Buffer, Term_Buffer) composited into final framebuffer with optional 2├ù scaling for sub-native resolutions
