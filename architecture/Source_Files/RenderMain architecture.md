# Subsystem Overview

## Purpose

RenderMain orchestrates the complete rendering pipeline for the Aleph One game engine, coordinating visibility determination, polygon depth-sorting, object placement, and rasterization to produce the on-screen image. Supports dual rendering backends (software rasterizer and OpenGL) and manages all visual effects including lighting, transfer modes, animated textures, and HUD overlays.

## Key Files

| File | Role |
|------|------|
| render.cpp / render.h | Core pipeline orchestrator; entry point for frame rendering; manages view state and effect application |
| RenderVisTree.cpp / RenderVisTree.h | Visibility culling via BSP ray-casting from viewpoint; constructs polygon visibility tree |
| RenderSortPoly.cpp / RenderSortPoly.h | Depth-sorts visible polygons into back-to-front rendering order; computes clipping windows |
| RenderPlaceObjs.cpp / RenderPlaceObjs.h | Places in-game objects (sprites, 3D models) into sorted rendering order with depth-based clipping |
| RenderRasterize.cpp / RenderRasterize.h | Transforms clipped polygons to screen space; delegates textured rendering to rasterizer backend |
| Rasterizer.h | Abstract base class defining rendering backend interface (Begin/End, texture calls) |
| Rasterizer_OGL.h | OpenGL implementation of rasterizer interface |
| Rasterizer_SW.h | Software implementation of rasterizer interface |
| OGL_Render.cpp / OGL_Render.h | OpenGL context lifecycle and coordinate transformation management |
| OGL_Setup.cpp / OGL_Setup.h | OpenGL detection, configuration, and asset loading orchestration |
| OGL_Textures.cpp / OGL_Textures.h | OpenGL texture allocation, loading, and lifecycle management with special effects (infravision, silhouette) |
| OGL_Model_Def.cpp / OGL_Model_Def.h | 3D model loading, transformation, and skin/texture management |
| OGL_Faders.cpp / OGL_Faders.h | OpenGL fade effect rendering with blend modes |
| scottish_textures.cpp / scottish_textures.h | Software texture rasterizer; polygon and rectangle texture-mapping |
| AnimatedTextures.cpp / AnimatedTextures.h | Animated wall texture frame sequences with per-sequence timing |
| shapes.cpp | Shape collection loading, RLE unpacking, and shading table generation |
| OGL_Subst_Texture_Def.cpp / OGL_Subst_Texture_Def.h | Substitute texture configuration and management for walls/sprites |
| OGL_Texture_Def.h | Base texture option structures for opacity and blending modes |
| ImageLoader.h / ImageLoader_SDL.cpp / ImageLoader_Shared.cpp | Image file loading (DDS, PNG, BMP) and format conversion (RGBA, DXTC) |
| Crosshairs.h / Crosshairs_SDL.cpp | Crosshair rendering and configuration |
| collection_definition.h | Data structure for asset collections (shapes, bitmaps, CLUTs) |
| shape_descriptors.h | Binary layout and macros for 16-bit shape descriptors (collection/shape/CLUT encoding) |
| shape_definitions.h | Collection header metadata and runtime pointers |
| textures.h / textures.cpp | Bitmap texture structures and row address precalculation |
| low_level_textures.h | Templated pixel-level texture rasterization with alpha blending |
| SW_Texture_Extras.cpp / SW_Texture_Extras.h | Software texture opacity table management |
| DDS.h | DirectDraw Surface file format structure definitions |

## Core Responsibilities

- Coordinate multi-stage rendering pipeline: visibility tree construction ΓåÆ polygon depth-sorting ΓåÆ object placement ΓåÆ coordinate transformation ΓåÆ rasterization
- Manage camera/view state (position, orientation, FOV, projection) and per-frame updates
- Implement two rendering backends (software rasterizer via template-based pixel writers; OpenGL via GPU)
- Perform BSP ray-casting visibility culling to determine visible polygons from viewpoint
- Sort polygons into back-to-front depth order with screen-space clipping window calculation
- Place game objects (sprites, 3D models) into sorted rendering order with distance-based scaling and projection
- Transform world-space coordinates to perspective-correct screen space with overflow handling for long distances
- Load, manage, and render textures across multiple formats (RGBA8, DXTC1/3/5, RLE-compressed sprites)
- Apply lighting via precalculated shading tables and per-pixel illumination lookup
- Render transfer modes (tinting, static effects, landscape textures, fade overlays) with platform-specific blending
- Manage animated wall textures with frame-sequencing and independent timing per animation
- Render first-person weapon/HUD layer and player crosshairs
- Support visual effects (earthquakes, fades, infravision tinting, silhouette rendering)
- Handle dual render path selection based on OpenGL availability

## Key Interfaces & Data Flow

**Inputs (consumed from other subsystems):**
- Game world geometry (map.h): polygon/endpoint/line/side data via accessor functions
- Player state (player.h): viewpoint position, orientation, current polygon, weapon
- Lighting (lightsource.h): per-polygon light intensities
- Media/liquids (media.h): liquid surface definitions and transparency
- Shape collections and bitmaps: indexed by shape descriptors with CLUT lookup
- Preferences (preferences.h): graphics mode, alpha blending options, rendering flags
- Rendering effects (ViewControl.h): FOV, landscape effects, tunnel vision state
- XML configuration: texture substitutions, 3D models, fog parameters

**Outputs (produced for display):**
- Rasterized 32-bit RGBA framebuffer to SDL surface or OpenGL backbuffer
- Screen updates via backend-specific swap/flush operations

**Data structures:**
- `view_data` ??? Camera state (position, orientation, FOV, screen dimensions, projection matrix)
- `polygon_definition` ??? Wall/floor/ceiling rendering data (texture, transfer mode, shading, lighting)
- `rectangle_definition` ??? Sprite/object rendering data (texture, scaling, positioning)
- `shape_descriptor` ??? Packed 16-bit index (5-bit collection + 8-bit shape + 3-bit CLUT)
- `bitmap_definition` ??? Bitmap pixel data with row address tables
- `clipping_window_data` ??? Screen-space rendering boundaries for polygons and objects
- `sorted_node_data` ??? Tree node in depth-sorted polygon list

## Runtime Role

**Initialization (per-frame):**
1. `render_view()` called from main game loop with current view_data
2. Memory allocated once via `allocate_render_memory()`
3. View parameters initialized via `initialize_view_data()`

**Per-frame rendering stages:**
1. Visibility tree construction via `RenderVisTreeClass` ??? BSP ray-casting determines visible polygons
2. Polygon depth-sorting via `RenderSortPolyClass` ??? converts tree to back-to-front order with clipping windows
3. Object placement via `RenderPlaceObjsClass` ??? projects 3D objects to screen space and integrates into sort order
4. Rasterization via `RenderRasterizerClass` ??? clips polygons to viewport, transforms to screen space, delegates to `RasterizerClass` backend
5. Backend execution ??? either `Rasterizer_SW_Class` (templated pixel writes) or `Rasterizer_OGL_Class` (OpenGL draw calls)
6. HUD/weapons layer rendered via `render_viewer_sprite_layer()`
7. Foreground effects applied (fade overlays, crosshairs, text via `OGL_RenderText()`)

**Shutdown:**
- Texture/model resources freed via OpenGL or software cleanup

## Notable Implementation Details

- **Fixed-point math:** Coordinates and dimensions use fixed-point representations with `FIXED_FRACTIONAL_BITS` precision; world space divided by `WORLD_ONE` (65536) for scaling
- **Overflow handling:** Long-distance rendering tracked via overflow flags in endpoint/line data to handle distances beyond 16-bit signed range
- **RLE sprite compression:** Shape collections use Marathon 1/2 RLE formats with endianness conversion; unpacked to SDL surfaces at load time
- **Template-based rasterization:** Low-level texture writing (`low_level_textures.h`) templated on bit-depth (8/16/32-bit) for compile-time specialization and performance
- **Dual backend abstraction:** `RasterizerClass` interface allows both software (immediate pixel writes) and OpenGL (vertex array/texture binding) implementations; selected at runtime based on `OGL_IsActive()`
- **Animated textures via frame sequences:** `AnimatedTextures` translates static shape descriptors to animated frame equivalents during render stage; per-sequence timing independent
- **Transfer modes:** Runtime instantiation of polygon/sprite rendering intent (textured, tinted, static/noise, landscape) affects blending and shader selection
- **Mipmaps and VRAM management:** OpenGL textures cached with age tracking; unused textures purged per type (walls: 10s, sprites: 15s, weapons: 20s); mipmap chain support via `gluBuild2DMipmaps` or `GL_SGIS_generate_mipmap`
- **Conditional OpenGL extensions:** ARB multitexture and compressed texture support dynamically loaded via `SDL_GL_GetProcAddress()` on Windows; fallback paths for unavailable extensions
- **Platform-specific details:** Mac Quickdraw context management via AGL; Win32 pixel format via WGL; SDL cross-platform; PSP support via PSPSDK libGU conditionally compiled
