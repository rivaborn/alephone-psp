# Source_Files/RenderMain/scottish_textures.h

## File Purpose
Defines core texture and surface rendering data structures for the Aleph One game engine (Marathon-based). Establishes contracts for rendering 2D rectangles (sprites/objects) and 3D polygons (walls/surfaces) with support for multiple transfer modes, shading tables, and both software and OpenGL rendering paths.

## Core Responsibilities
- Define transfer modes (tinting, solid, textured, shadeless, static) for texture rendering
- Define tint table structures for 8, 16, and 32-bit color depths
- Declare `rectangle_definition` struct for sprite/object rendering (weapons, items, scenery)
- Declare `polygon_definition` struct for wall/surface polygon rendering
- Expose shading table management state and initialization entry point
- Support both 2D (software rasterizer) and 3D (OpenGL model) rendering paths

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tint_table8` | struct | 8-bit indexed color tinting table (256 entries) |
| `tint_table16` | struct | 16-bit RGB component tinting tables |
| `tint_table32` | struct | 32-bit RGB component tinting tables |
| `rectangle_definition` | struct | Renderable rectangle (sprite/object): screen coords, depth, shading, texture, OpenGL 3D model data |
| `polygon_definition` | struct | Renderable convex polygon (wall): vertices, shading, 3D origin/vector, texture |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bit_depth` | `short` | extern | Current rendering color depth |
| `interface_bit_depth` | `short` | extern | UI/HUD rendering color depth |
| `number_of_shading_tables` | `short` | extern | Count of available shading tables |
| `shading_table_fractional_bits` | `short` | extern | Fractional bits in shading interpolation |
| `shading_table_size` | `short` | extern | Byte size of single shading table |

## Key Functions / Methods

### allocate_texture_tables
- **Signature:** `void allocate_texture_tables(void)`
- **Purpose:** Initialize and allocate memory for shading and tinting tables
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates global shading/tinting table memory; populates shading state globals
- **Calls:** (defined elsewhere, not visible here)
- **Notes:** Called once during engine initialization

## Control Flow Notes
This file is a **data contract** layer between game logic and rendering subsystems:
- Game code populates `rectangle_definition` (for sprites, weapons) and `polygon_definition` (for walls)
- Software rasterizer (`Rasterizer_SW`) and OpenGL renderer consume these structures
- Texture lookup, shading, and transfer mode application happen during rasterization

## External Dependencies
- **Includes:** `shape_descriptors.h` (defines `shape_descriptor` typedef; collection enums)
- **Forward declared:** `OGL_ModelData` (OpenGL 3D model container, defined elsewhere)
- **Undefined types used:** `pixel8`, `pixel16`, `pixel32` (color formats, likely from pixel-format header); `_fixed` (fixed-point); `world_point3d`, `world_vector3d`, `point2d` (3D math); `GLfloat` (OpenGL)
- **Referenced elsewhere:** Rendering code, shape/collection management, OpenGL subsystem
