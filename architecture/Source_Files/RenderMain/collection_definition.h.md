# Source_Files/RenderMain/collection_definition.h

## File Purpose
Defines core data structures for asset collections in the rendering system. Collections are containers holding shape animations, individual frames/bitmaps, and color lookup tables (CLUTs) used throughout the game engine.

## Core Responsibilities
- Define `collection_definition` structure as the primary container for all shape and bitmap assets
- Specify `high_level_shape_definition` for animation metadata (views, frames, transfer modes, sounds)
- Specify `low_level_shape_definition` for individual bitmap frames with rendering parameters (mirroring, lighting, origin/keypoint positions)
- Specify `rgb_color_value` structure for color table entries
- Define collection type enumeration (_wall_collection, _object_collection, _interface_collection, _scenery_collection)
- Provide versioning, size constants, and bit-flag definitions for rendering parameters

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| collection_definition | struct | Container holding all assets: color tables, high-level shapes (animations), low-level shapes (frames), and bitmaps. Includes metadata like version, type, pixel-to-world scaling. |
| high_level_shape_definition | struct | Animation metadata: number of views, frames per view, timing, transfer modes, keyframe sounds, and indices into low-level shapes. |
| low_level_shape_definition | struct | Single bitmap frame: rendering flags (mirroring, keypoint obscured, lighting), bitmap index, pixel/world coordinates (origin, keypoint, bounds), world position. |
| rgb_color_value | struct | Color entry: flags (self-luminescent), value field, and RGB components. |
| Collection types (enum) | enum | _unused_collection, _wall_collection, _object_collection, _interface_collection, _scenery_collection |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| COLLECTION_VERSION | macro (int) | global | Version 3; marks structure layout expectations |
| NUMBER_OF_PRIVATE_COLORS | macro (int) | global | Reserved color slots in CLUT (3) for engine use |
| _X_MIRRORED_BIT, _Y_MIRRORED_BIT, _KEYPOINT_OBSCURED_BIT | macros (uint16) | global | Bit flags in low_level_shape_definition.flags (bits 15, 14, 13) |
| SELF_LUMINESCENT_COLOR_FLAG | macro (uint8) | global | Bit flag in rgb_color_value.flags (bit 7) |
| SIZEOF_collection_definition, SIZEOF_high_level_shape_definition, SIZEOF_low_level_shape_definition, SIZEOF_rgb_color_value | macros (int) | global | Binary layout sizes for serialization (544, 90, 36, 8 bytes) |

## Key Functions / Methods
None. This is a pure definition header with no function declarations.

## Control Flow Notes
This file provides data structure definitions used at multiple engine stages:
- **Asset loading**: Collection structures deserialized from disk (collection_definition size used for offset assertion)
- **Animation**: high_level_shape_definition drives frame selection and sound triggering
- **Rendering**: low_level_shape_definition accessed per frame for bitmap, mirroring, lighting, and positioning
- **Color management**: rgb_color_value arrays indexed via low_level_shape_definition.bitmap_index

## External Dependencies
- **Standard library**: `<vector>` (used for dynamic arrays in collection_definition)
- **Forward declarations**: `struct bitmap_definition`, `struct high_level_shape_definition`, `struct low_level_shape_definition`, `struct rgb_color_value`
- **Referenced but not defined**: `_fixed` type (likely defined in a separate header), `shape_animation_data` (mentioned in comment as interface.h definition)

---

**Notes:**
- `high_level_shape_definition.low_level_shape_indexes[1]` is a flexible array member; actual count depends on `number_of_views ├ù frames_per_view` (see interface.h/shape_animation_data).
- `pixels_to_world` appears in both collection_definition and high_level_shape_definition, allowing per-shape override of coordinate scaling.
- Bit flags in low_level_shape_definition encode rendering intent (mirroring, keypoint visibility) and lighting requirements in a single uint16.
