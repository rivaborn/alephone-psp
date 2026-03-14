# Source_Files/RenderMain/render.h

## File Purpose

Header for the core rendering system of the Aleph One game engine (Marathon-compatible). Defines the main view/camera data structure, rendering state management, visibility flags for BSP culling, and prototypes for frame rendering, effects, and UI overlays.

## Core Responsibilities

- Define `view_data` struct containing all camera/viewport state (position, orientation, FOV, screen dimensions, projection parameters)
- Provide visibility culling flags and macros for portal/BSP traversal (polygon/endpoint/side visibility bits)
- Declare main render entry point (`render_view`) and frame-setup functions
- Manage rendering effects (fold-in/out, explosions, tunnel vision state)
- Provide overhead map and computer terminal UI rendering
- Support transfer mode instantiation for textured surfaces (tinting, static, landscape effects)
- Track view modes (terminal, overhead map, tunnel vision, media boundary)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct | Complete camera/viewport state: FOV, position, rotation, screen metrics, projection parameters, effect state, media tracking |
| `point2d` | struct | 2D screen coordinates (x, y as shorts) |
| `definition_header` | struct | Generic header for definitions (tag, clip bounds) |
| Render flags (enums) | enum / bitflags | Visibility state for culling: polygon/endpoint/side visibility, clip data presence, transformation state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RenderFlagList` | `vector<uint16>` | global | Resizable array of visibility/state flags for polygons, lines, endpoints, and sides during BSP traversal |

## Key Functions / Methods

### allocate_render_memory
- Signature: `void allocate_render_memory(void)`
- Purpose: Initialize and allocate memory for render system structures
- Inputs: None
- Outputs: None (side effect allocation)
- Side effects: Allocates global rendering buffers

### initialize_view_data
- Signature: `void initialize_view_data(struct view_data *view)`
- Purpose: Initialize a view_data structure with default or derived values (called during setup or when view parameters change)
- Inputs: Pointer to `view_data` struct to initialize
- Outputs: None (struct modified in-place)
- Side effects: Populates screen scaling, projection cone angles, edge vectors based on FOV and screen dimensions
- Notes: Must be called before render_view; derives values like half_screen_width, world_to_screen scaling factors

### render_view
- Signature: `void render_view(struct view_data *view, struct bitmap_definition *destination)`
- Purpose: Render one frame of the game world into a destination bitmap
- Inputs: Current `view_data` state; destination bitmap to render into
- Outputs: None (bitmap modified in-place)
- Side effects: Modifies destination bitmap; may update view tick counters
- Calls: (implementation in RENDER.C, not visible in header)
- Notes: Main per-frame rendering call; uses view_data to determine camera, FOV, effects, visibility state

### start_render_effect
- Signature: `void start_render_effect(struct view_data *view, short effect)`
- Purpose: Initiate a visual effect (e.g., teleport fold, explosion) that will animate over subsequent frames
- Inputs: `view_data` to modify; effect ID (e.g., `_render_effect_fold_in`)
- Outputs: None (view_data.effect and effect_phase modified)
- Side effects: Sets effect and effect_phase in view_data

### render_overhead_map
- Signature: `void render_overhead_map(struct view_data *view)`
- Purpose: Render the tactical overhead map onto the current frame
- Inputs: Current view_data
- Outputs: None (modifies destination bitmap)

### render_computer_interface
- Signature: `void render_computer_interface(struct view_data *view)`
- Purpose: Render terminal/computer UI on top of game view
- Inputs: Current view_data
- Outputs: None

### instantiate_rectangle_transfer_mode
- Signature: `void instantiate_rectangle_transfer_mode(view_data *view, rectangle_definition *rectangle, short transfer_mode, _fixed transfer_phase)`
- Purpose: Apply a transfer mode effect to a screen rectangle (tinting, static, etc.)
- Inputs: view, rectangle definition, transfer mode ID, phase/time offset
- Outputs: None (rectangle modified or rendered with effect applied)

### instantiate_polygon_transfer_mode
- Signature: `void instantiate_polygon_transfer_mode(view_data *view, polygon_definition *polygon, short transfer_mode, bool horizontal)`
- Purpose: Apply a transfer mode effect to a world polygon
- Inputs: view, polygon definition, transfer mode ID, horizontal orientation flag
- Outputs: None

### ResetOverheadMap
- Signature: `void ResetOverheadMap(void)` (in overhead_map.cpp)
- Purpose: Clear and reinitialize overhead map state
- Inputs: None
- Outputs: None

## Control Flow Notes

This file anchors the **render loop**: `initialize_view_data()` is called during setup or when camera parameters change; `render_view()` is the per-frame rendering call that traverses the BSP and rasterizes geometry. Effects initiated by `start_render_effect()` animate the `view_data.effect_phase` field across frames, modulating the view (e.g., tunnel vision narrowing, fold-in animation). Visibility flags in `render_flags` are cleared and repopulated each frame to mark which polygons, endpoints, and sides are within the view cone and should be drawn. Overhead map and terminal UI are rendered as overlays on top of the main view.

## External Dependencies

- **world.h** ΓÇö `world_point3d`, `world_vector2d/3d`, `world_distance`, angle types, trig tables, coordinate transforms
- **textures.h** ΓÇö `bitmap_definition` struct for surface textures
- **ViewControl.h** ΓÇö Field-of-view accessors (`View_FOV_Normal()`, etc.), landscape and effect control flags
- **scottish_textures.h** ΓÇö `rectangle_definition`, `polygon_definition`, transfer mode constants, shape descriptors
