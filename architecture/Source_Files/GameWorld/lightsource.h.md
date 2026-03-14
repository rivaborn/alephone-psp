# Source_Files/GameWorld/lightsource.h

## File Purpose
Header defining the lighting system for the game engine. Declares light data structures, state management, and animation parameters, supporting both modern and legacy (Marathon 1) light formats for backward compatibility.

## Core Responsibilities
- Define static (template) and dynamic (runtime) light data structures
- Specify light types (normal, strobe, media) and state transitions
- Define lighting transition functions (constant, linear, smooth, flicker)
- Provide flag macros for light configuration
- Declare light management, query, and frame-update functions
- Support serialization/deserialization of light data
- Maintain backward compatibility with Marathon 1 map format

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lighting_function_specification` | struct | Defines light intensity transition: function type, period, initial/final intensity (14 bytes) |
| `static_light_data` | struct | Immutable light template: type, flags, phase, six transition specs for all state pairs (100 bytes) |
| `light_data` | struct | Runtime light state: flags, current state, calculated intensity, phase/period, initial/final intensity (128 bytes) |
| `old_light_data` | struct | Legacy Marathon 1 format: flags, type, mode, intensity, period (32 bytes) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LightList` | `vector<light_data>` | extern global | Master list of all active lights in the current map |
| `lights` | macro (pointer) | global | Convenience alias to first element of LightList |
| `MAXIMUM_LIGHTS_PER_MAP` | macro (size_t) | global | Current count of lights; updates when lights are added |

## Key Functions / Methods

### new_light
- Signature: `short new_light(struct static_light_data *data)`
- Purpose: Create and register a new light in the map
- Inputs: Pointer to static light template data
- Outputs/Return: Index/ID of newly created light
- Side effects: Appends to LightList

### update_lights
- Signature: `void update_lights(void)`
- Purpose: Per-frame update: advance light state machines and recalculate intensity using lighting function specifications
- Side effects: Updates `intensity` and `phase` fields in all lights in LightList
- Notes: Called once per frame; each light transitions through states using its 6 lighting_function_specifications

### get_light_status / set_light_status
- Signature: `bool get_light_status(size_t light_index)` / `bool set_light_status(size_t light_index, bool active)`
- Purpose: Query or change active/inactive state of a light
- Inputs: Light index; new state (set only)
- Outputs/Return: Current active status; success flag (set only)
- Side effects: (set) Triggers light state transition

### set_tagged_light_statuses
- Signature: `bool set_tagged_light_statuses(short tag, bool new_status)`
- Purpose: Bulk update lights matching a tag (e.g., all lights on a platform switch)
- Inputs: Tag value, new state
- Outputs/Return: Success status
- Side effects: Updates state for all lights with matching tag

### get_light_intensity / get_light_data
- Signature: `_fixed get_light_intensity(size_t light_index)` / `light_data *get_light_data(const size_t light_index)`
- Purpose: Query current light intensity or full runtime state
- Inputs: Light index
- Outputs/Return: Intensity (fixed-point) or pointer to light_data struct
- Notes: Accessors for rendering or game logic queries

## Control Flow Notes
- **Init**: Maps load lights via `unpack_*_light_data` functions; legacy M1 formats converted via `convert_old_light_data_to_new`
- **Per-frame**: `update_lights()` advances state machines and applies lighting function curves
- **State machine**: Each light transitions through six states: _becoming_active, _primary_active, _secondary_active, _becoming_inactive, _primary_inactive, _secondary_inactive
- **Lighting curves**: Each state pair has a transition function (constant, linear, smooth sine, flicker) controlling intensity
- **Shutdown/Save**: `pack_*_light_data` functions serialize lights back to map file

## External Dependencies
- `#include <vector>` ΓÇô STL vector for LightList
- Custom types: `_fixed` (fixed-point), `int16`, `uint16`, `uint8`, `size_t`
- Macro utilities: `TEST_FLAG16`, `SET_FLAG16` (defined elsewhere)

**Notes**: Serialization helpers (`pack_old_light_data`, `unpack_static_light_data`, etc.) handle map I/O; `get_defaults_for_light_type` provides factory configs.
