# Source_Files/GameWorld/map_constructors.cpp

## File Purpose
Constructs and initializes map geometry elements (polygons, lines, sides, endpoints) during map loading and editing. Recalculates derived geometric properties and precalculates collision/neighbor data. Provides binary serialization (pack/unpack) for saving and loading map state.

## Core Responsibilities
- **Geometry creation**: Create new sides, endpoints, lines, polygons with proper ownership and linkage
- **Data recalculation**: Recompute derived properties (area, endpoints, adjacent polygons, heights, lightsources)
- **Redundancy elimination**: Calculate values that can be inferred from geometry but are cached for performance
- **Collision preprocessing**: Build exclusion zones and neighbor lists via flood-fill for runtime impassability checks
- **Serialization**: Pack/unpack all map data types to/from byte streams (big-endian format)
- **Lightsource assignment**: Infer which light sources illuminate polygon sides based on geometry type

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `intersecting_flood_data` | struct | Temporary data for flood-fill searches to find nearby endpoints/lines/polygons within distance threshold |
| `polygon_data` | struct (defined in map.h) | Core polygon with vertices, textures, heights, exclusion zones, neighbors |
| `side_data` | struct (defined in map.h) | Wall segment with textures, transfer modes, control panels, lightsources |
| `line_data` | struct (defined in map.h) | Edge segment between two endpoints; references polygon owners and sides |
| `endpoint_data` | struct (defined in map.h) | Vertex with elevation and transparency flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `map_index_buffer_count` | `long` | static | Size of the map index buffer (set dynamically; used to validate bounds) |
| `LineIndices` | `vector<short>` | static | Temporary list of line indices found during flood operations (reused per search) |
| `EndpointIndices` | `vector<short>` | static | Temporary list of endpoint indices found during flood operations |
| `PolygonIndices` | `vector<short>` | static | Temporary list of polygon indices found during flood operations |
| `DoIncorrectCountVWarn` | `const bool` | global | Controls whether to emit warnings when index lists overflow (max 64) |

## Key Functions / Methods

### recalculate_side_type
- **Signature**: `void recalculate_side_type(short side_index)`
- **Purpose**: Determine side type (_full_side, _high_side, _low_side, _split_side) based on ceiling/floor heights of adjacent polygons
- **Inputs**: Side index into SideList
- **Outputs/Return**: None; updates side->type
- **Side effects**: Modifies side_data in SideList
- **Calls**: `get_side_data()`, `find_adjacent_polygon()`, `get_polygon_data()`
- **Notes**: Implements visibility logic for wall texturesΓÇöfull sides are walls between rooms of different heights; high/low sides are partial walls

### new_side
- **Signature**: `short new_side(short polygon_index, short line_index)`
- **Purpose**: Create a new side attached to a polygon and line; initialize default textures and update polygon's side list
- **Inputs**: Polygon and line indices
- **Outputs/Return**: New side index
- **Side effects**: Appends to SideList, increments dynamic_world->side_count, links side to polygon and line
- **Calls**: `get_line_data()`, `get_polygon_data()`, `recalculate_redundant_side_data()`, `calculate_adjacent_sides()`, `recalculate_side_type()`
- **Notes**: Asserts that line is not already assigned to this polygon; initializes textures to UNONE

### recalculate_redundant_polygon_data
- **Signature**: `void recalculate_redundant_polygon_data(short polygon_index)`
- **Purpose**: Recompute cached polygon properties: endpoint list, adjacent polygons, area, center, sides
- **Inputs**: Polygon index
- **Outputs/Return**: None; updates polygon_data in PolygonList
- **Side effects**: Modifies polygon's endpoint_indexes, adjacent_polygon_indexes, area, center, side_indexes
- **Calls**: `calculate_clockwise_endpoints()`, `calculate_adjacent_polygons()`, `calculate_polygon_area()`, `find_center_of_polygon()`, `calculate_adjacent_sides()`
- **Notes**: Skips detached (shadow) polygons; leaves lightsource fields unset (marked TODO)

### recalculate_redundant_endpoint_data
- **Signature**: `void recalculate_redundant_endpoint_data(short endpoint_index)`
- **Purpose**: Compute endpoint solidity, transparency, elevation flags and adjacent height extrema
- **Inputs**: Endpoint index
- **Outputs/Return**: None; updates endpoint_data in EndpointList
- **Side effects**: Sets flags and height bounds for collision/rendering
- **Calls**: `get_endpoint_data()`, `get_line_data()`, `get_polygon_data()` (iterates all lines containing endpoint)
- **Notes**: Scans all lines in the map to find those touching this endpoint; solid if any touching line is solid

### recalculate_redundant_line_data
- **Signature**: `void recalculate_redundant_line_data(short line_index)`
- **Purpose**: Recompute line length, adjacent ceiling/floor bounds, elevation/transparenc flags
- **Inputs**: Line index
- **Outputs/Return**: None; updates line_data in LineList
- **Side effects**: Recalculates redundant_side_data for both sides of the line
- **Calls**: `get_line_data()`, `get_endpoint_data()`, `get_polygon_data()`, `recalculate_redundant_side_data()`
- **Notes**: Marks elevation if adjacent polygons have different floor heights; marks variable_elevation for platform polygons

### precalculate_map_indexes
- **Signature**: `void precalculate_map_indexes(void)`
- **Purpose**: Build collision exclusion zones and neighbor lists for all non-detached polygons
- **Inputs**: None (operates on global map state)
- **Outputs/Return**: None; populates map_indexes buffer and updates polygon's exclusion_zone and neighbor fields
- **Side effects**: Calls flood_map twice per polygon; appends to MapIndexList
- **Calls**: `find_intersecting_endpoints_and_lines()` (2x per polygon), `precalculate_polygon_sound_sources()`
- **Notes**: Two passes with different separation distances: MINIMUM_SEPARATION_FROM_WALL and MINIMUM_SEPARATION_FROM_PROJECTILE; used for runtime collision testing

### find_intersecting_endpoints_and_lines
- **Signature**: `static void find_intersecting_endpoints_and_lines(short polygon_index, world_distance minimum_separation)`
- **Purpose**: Flood-fill from polygon to find all endpoints and lines within distance threshold; populate global index vectors
- **Inputs**: Source polygon, separation distance (squared internally)
- **Outputs/Return**: None; clears and populates LineIndices, EndpointIndices, PolygonIndices
- **Side effects**: Global state modified; vectors are cleared at start
- **Calls**: `find_center_of_polygon()`, `flood_map()` via `intersecting_flood_proc()`
- **Notes**: Uses flood-fill to traverse adjacent polygons; callback procedure tests point-to-line distances

### guess_side_lightsource_indexes
- **Signature**: `void guess_side_lightsource_indexes(short side_index)`
- **Purpose**: Assign light source indices to a side based on its type and polygon configuration
- **Inputs**: Side index
- **Outputs/Return**: None; sets primary, secondary, transparent lightsource indices
- **Side effects**: Modifies side_data in SideList
- **Calls**: `get_side_data()`, `get_line_data()`, `get_polygon_data()`
- **Notes**: Full sides use ceiling light; split sides choose based on gap height; high sides use ceiling, low sides use floor

### Packing / Unpacking Functions
- **Signatures**: `uint8 *unpack_*_data(uint8 *Stream, T *Objects, size_t Count)` and symmetric `pack_*_data` versions
- **Purpose**: Convert between big-endian byte stream and native game data structures
- **Inputs**: Byte stream pointer, object array, count
- **Outputs/Return**: Updated stream pointer (advanced by serialized size)
- **Side effects**: Modifies memory at Objects
- **Calls**: StreamToValue/ValueToStream macros, StreamToList/ListToStream, etc. from Packing.h
- **Data types**: endpoint_data, line_data, side_data, polygon_data, map_annotation, map_object, dynamic_data, object_data, damage_definition, etc.
- **Notes**: All asserts verify correct byte count consumed; supports nested structures (e.g., polygon's arrays of indices)

### Helper Functions (summarized under Notes)
- `calculate_clockwise_endpoints()`, `calculate_adjacent_polygons()`, `calculate_adjacent_sides()`: Static helpers to iterate polygon geometry
- `calculate_polygon_area()`: Shoelace formula for 2D polygon area
- `add_map_index()`: Appends index to MapIndexList and increments count
- `set_map_index_buffer_size()`: Stores buffer size for validation

## Control Flow Notes
- **Map loading/generation phase**: `recalculate_redundant_*_data()` functions are called by map editor and loaders to rebuild derived state after geometry is modified
- **Game startup**: `precalculate_map_indexes()` is called once per level to precompute collision zones (called during `initialize_map_for_new_level()`)
- **Save/load operations**: Pack functions write map snapshots; unpack functions restore from disk
- **Runtime**: Exclusion zones and neighbor lists are read-only; used by collision detection and pathfinding

## External Dependencies
- **Includes**: `cseries.h` (basic types, macros), `map.h` (data structures), `flood_map.h` (flood_map function), `Packing.h` (serialization macros)
- **Global arrays** (defined elsewhere): `SideList`, `LineList`, `EndpointList`, `PolygonList`, `MapIndexList` (all std::vector<>)
- **Global pointers**: `static_world` (map metadata), `dynamic_world` (runtime counts and state)
- **Utility functions** (defined elsewhere): `get_side_data()`, `get_line_data()`, `get_polygon_data()`, `get_endpoint_data()` (accessors), `find_center_of_polygon()`, `flood_map()`, `clockwise_endpoint_in_line()`, `find_adjacent_polygon()`, `push_out_line()`, `distance2d()`, `obj_clear()`, `find_center_of_polygon()`
- **Macros from map.h**: `POLYGON_IS_DETACHED()`, `LINE_IS_SOLID()`, `LINE_IS_TRANSPARENT()`, `LINE_IS_ELEVATION()`, `SET_*` flag operations
