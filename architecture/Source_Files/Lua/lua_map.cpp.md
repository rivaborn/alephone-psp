# Source_Files/Lua/lua_map.cpp

## File Purpose
Implements Lua C bindings for Marathon/Aleph One game world map data. Registers map geometry (polygons, lines, endpoints, sides), dynamic entities (platforms, lights, media/liquids), and map metadata (fog, level properties, annotations) as Lua userdata with getter/setter functions, enabling Lua scripts to query and modify the game world at runtime.

## Core Responsibilities
- Register all map-related Lua classes (geometry, platforms, lights, sides, control panels, media) with Lua interpreter
- Implement property accessors (getters/setters) for map objects and their attributes
- Bridge C++ game world state with Lua scripting environment
- Provide container-style access to iterate over game world elements (lines, polygons, platforms, lights, etc.)
- Load backwards-compatibility Lua functions for deprecated APIs
- Handle type conversions between Lua and C++ (world coordinates, bitmasks, enum values)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `control_panel_definition` | struct | Local definition of control panel properties (collection, active/inactive shapes, sounds) |
| `L_Class<name>` | template class (from lua_templates.h) | Wraps individual indexed objects with Lua metatable and getter/setter registry |
| `L_Enum<name>` | template class (from lua_templates.h) | Wraps enumerated types with mnemonic lookup and equality support |
| `L_Container<name, T>` | template class (from lua_templates.h) | Provides table-like iteration and indexed access to object collections |
| `L_EnumContainer<name, T>` | template class (from lua_templates.h) | Container that supports string-based mnemonic lookups for enum collections |
| `Lua_Polygon_Floor` | L_Class typedef | Exposes floor properties (height, texture, light, transfer mode) |
| `Lua_Polygon_Ceiling` | L_Class typedef | Exposes ceiling properties (height, texture, light, transfer mode) |
| `Lua_Platform` | L_Class typedef | Exposes platform state (active, height, controllability, speed, direction) |
| `Lua_Annotation` | L_Class typedef | Wraps map annotations (text labels with position and polygon) |
| `Lua_Fog` / `Lua_Fog_Color` | L_Class typedefs | Exposes fog rendering parameters (depth, color, presence) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Collection_Name` | `char[]` | static | Metatable name for collection enum class |
| `Lua_Collections_Name` | `char[]` | static | Global Lua table name for collections container |
| Various `Lua_*_Name` | `char[]` | static | Metatable and container names for 30+ map entity types |
| `compatibility_script` | `const char*` | static | Embedded Lua code defining legacy API wrapper functions (backwards compatibility) |
| `control_panel_definition` struct enum | unnamed enum | static | Local definition of control panel sound indices (activating, deactivating, unusable) |

## Key Functions / Methods

Most functions follow the pattern `Lua_<Type>_Get_<Property>` or `Lua_<Type>_Set_<Property>`, taking `lua_State *L` and returning `int` (Lua stack values). Examples listed below; 100+ similar accessor functions exist.

### Lua_Map_register
- **Signature:** `int Lua_Map_register(lua_State *L)`
- **Purpose:** Main registration function; called during engine initialization to expose all map-related classes to Lua
- **Inputs:** Lua state
- **Outputs/Return:** 0 (Lua stack effects are the registration side effects)
- **Side effects (global state, I/O, alloc):** Registers ~30 Lua classes and containers; sets up metatables and getter/setter registries in Lua registry
- **Calls (direct calls visible in this file):** All `Lua_*::Register()` template methods; `compatibility()` to load legacy functions
- **Notes:** Called once at startup; sets up two-way bindings between C++ map data and Lua environment

### Lua_Platform_Get_Active
- **Signature:** `static int Lua_Platform_Get_Active(lua_State *L)`
- **Purpose:** Expose platform active/inactive state to Lua
- **Inputs:** Lua state with platform object at stack index 1
- **Outputs/Return:** 1 (boolean on stack)
- **Side effects:** None (read-only)
- **Calls:** `Lua_Platform::Index()`, `get_platform_data()`, `PLATFORM_IS_ACTIVE()` macro
- **Notes:** Similar getter pattern used for all platform properties

### Lua_Platform_Set_Active
- **Signature:** `static int Lua_Platform_Set_Active(lua_State *L)`
- **Purpose:** Allow Lua to activate/deactivate platforms
- **Inputs:** Lua state with platform object at index 1, boolean at index 2
- **Outputs/Return:** 0 (no return value)
- **Side effects (global state, I/O, alloc):** Calls `set_platform_state()` to modify platform physics state
- **Calls:** `Lua_Platform::Index()`, `set_platform_state()`
- **Notes:** Validates argument type before calling game logic

### Lua_Platform_Set_Floor_Height / Lua_Platform_Set_Ceiling_Height
- **Signature:** `static int Lua_Platform_Set_Floor_Height(lua_State *L)` (and ceiling variant)
- **Purpose:** Allow Lua scripts to dynamically change platform geometry
- **Inputs:** Platform index + new height (in world units, divided by WORLD_ONE for Lua)
- **Outputs/Return:** 0
- **Side effects:** Modifies polygon heights; recalculates endpoint and line geometry; adjusts for media
- **Calls:** `adjust_platform_endpoint_and_line_heights()`, `adjust_platform_for_media()`, `recalculate_redundant_*_data()`
- **Notes:** Complex geometry updates; critical for platform movement

### Lua_Polygon_Floor_Set_Height / Lua_Polygon_Ceiling_Set_Height
- **Signature:** `static int Lua_Polygon_Floor_Set_Height(lua_State *L)` (and ceiling variant)
- **Purpose:** Allow terrain/ceiling deformation via Lua
- **Inputs:** Polygon index + new height
- **Outputs/Return:** 0
- **Side effects:** Modifies polygon height; iterates over all vertices to recalculate redundant data
- **Calls:** `recalculate_redundant_endpoint_data()`, `recalculate_redundant_line_data()` (called for each vertex)
- **Notes:** Loops over vertex_count to update all connected geometry

### Lua_Polygon_Floor_Set_Collection / Lua_Polygon_Floor_Set_Texture_Index
- **Signature:** `static int Lua_Polygon_Floor_Set_Collection(lua_State *L)` (and texture variant)
- **Purpose:** Change floor/ceiling texture collection or shape index at runtime
- **Inputs:** Polygon index + collection/shape index
- **Outputs/Return:** 0
- **Side effects:** Modifies polygon texture descriptors (repacked via `BUILD_DESCRIPTOR()`)
- **Calls:** `Lua_Collection::ToIndex()` or direct cast
- **Notes:** Validates shape index against `MAXIMUM_SHAPES_PER_COLLECTION`

### Lua_Sides_New
- **Signature:** `static int Lua_Sides_New(lua_State *L)`
- **Purpose:** Constructor; create new side/wall between a polygon and line
- **Inputs:** Polygon index (number or Lua_Polygon object) + line index
- **Outputs/Return:** 1 (new Lua_Side object on stack)
- **Side effects:** Calls `new_side()` to allocate geometry; returns NONE if side already exists
- **Calls:** `Lua_Polygon::Valid()`, `Lua_Line::Valid()`, `get_line_data()`, `new_side()`
- **Notes:** Validates polygon-line relationship; errors if side already exists or line not shared by polygon

### Lua_Tag_Get_Active / Lua_Tag_Set_Active
- **Signature:** `static int Lua_Tag_Get_Active(lua_State *L)` / `static int Lua_Tag_Set_Active(lua_State *L)`
- **Purpose:** Query/modify active state of all lights and platforms sharing a tag
- **Inputs:** Tag index + (for Set) boolean state
- **Outputs/Return:** 1 (boolean) / 0
- **Side effects (global state, I/O, alloc):** Iterates all lights and platforms to check/set tag-matched states
- **Calls:** `get_light_status()`, `PLATFORM_IS_ACTIVE()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `assume_correct_switch_position()`
- **Notes:** Linear scan over all lights and platforms (no index); important for puzzle/gameplay logic

### Lua_Annotations_New
- **Signature:** `static int Lua_Annotations_New(lua_State *L)`
- **Purpose:** Create new map annotation (text label)
- **Inputs:** Polygon (optional) + text + x, y (optional coordinates)
- **Outputs/Return:** 1 (new Lua_Annotation object)
- **Side effects:** Appends to `MapAnnotationList`; increments `dynamic_world->default_annotation_count`
- **Calls:** `MapAnnotationList.push_back()`, `Lua_Annotation::Push()`
- **Notes:** Defaults position to polygon center if not specified; caps text at `MAXIMUM_ANNOTATION_TEXT_LENGTH`

### compatibility
- **Signature:** `static void compatibility(lua_State *L)`
- **Purpose:** Load embedded Lua compatibility script to support old map API
- **Inputs:** Lua state
- **Outputs/Return:** void
- **Side effects:** Executes embedded Lua code via `luaL_loadbuffer()` + `lua_pcall()`; defines 40+ wrapper functions
- **Calls:** `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** `compatibility_script` is a massive string of Lua function definitions on a single line (minified); invoked at registration time

**Notes on Remaining Functions:**
All remaining functions (`Lua_Line_Get_*`, `Lua_Light_Get/Set_*`, `Lua_Media_Get_*`, `Lua_Endpoint_Get_*`, `Lua_Fog_Get/Set_*`, etc.) follow identical patterns: extract object index from Lua state, call C++ accessor functions, push result back to Lua, and handle type validation.

## Control Flow Notes

**Initialization:** `Lua_Map_register()` is called once during engine startup (location not visible in this file, but typical for Lua binding code). It registers all classes and containers, then injects backwards-compatibility functions.

**Runtime Operation:** Lua scripts call registered functions (e.g., `Polygons[i].floor.height = 256`), which invoke getter/setter functions. These functions:
1. Extract C++ indices from Lua userdata
2. Call game world accessor/mutator functions (e.g., `get_platform_data()`, `set_light_status()`)
3. Push results back to Lua

**Update Loop Integration:** Not directly visible in this file. Map changes made via Lua are applied immediately to global `static_world`, `dynamic_world`, and various object lists. The game loop (outside this file) reads these modified structures each frame.

**Backwards Compatibility:** The `compatibility_script` wraps new Lua API calls into legacy function names, allowing old scripts to work unchanged.

## External Dependencies

**Lua C API:**
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô stack manipulation, error handling, registry access

**Game World Data:**
- `map.h` ΓÇô geometry structures (`polygon_data`, `line_data`, `endpoint_data`, `side_data`, `object_data`), global lists (`PolygonList`, `LineList`, etc.), and accessor macros
- `lightsource.h` ΓÇô light structures and accessors (`LightList`, `get_light_data()`)
- `media.h` ΓÇô media/liquid structures (`MediaList`, `get_media_data()`)
- `platforms.h` ΓÇô platform structures and state functions (`PlatformList`, `get_platform_data()`, `set_platform_state()`)
- `collection_definition.h` ΓÇô collection metadata (forward declared, `get_collection_definition()`)

**Rendering/Engine:**
- `OGL_Setup.h` ΓÇô fog data retrieval (`OGL_GetFogData()`, `OGL_Fog_AboveLiquid`, `OGL_Fog_BelowLiquid`)
- `SoundManager.h` ΓÇô audio system
- `lua_monsters.h`, `lua_objects.h`, `lua_templates.h` ΓÇô other Lua binding modules

**External State/Functions (defined elsewhere, called here):**
- Global mutable state: `static_world`, `dynamic_world`, `MapAnnotationList`, various `*List` containers
- Functions: `get_light_status()`, `set_light_status()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `assume_correct_switch_position()`, `adjust_platform_for_media()`, `recalculate_redundant_*_data()`, `new_side()`, `number_of_terminal_texts()`
- Macros: `PLATFORM_IS_ACTIVE()`, `LINE_IS_SOLID()`, `GET_DESCRIPTOR_SHAPE()`, `BUILD_DESCRIPTOR()`, etc.
