# Source_Files/GameWorld/lightsource.cpp

## File Purpose
Implements dynamic light source management for the Aleph One game engine (Marathon series). Manages creation, animation, and state transitions of in-game lights with support for multiple lighting types and animation functions. Handles both runtime light updates and serialization/deserialization of light data.

## Core Responsibilities
- Create and manage light instances in `LightList` vector
- Update light intensity each frame based on lighting function and animation phase
- Handle light state transitions (becoming active/inactive, primary/secondary states)
- Manage stateless (six-phase) lights vs. state-machine lights
- Query and modify light status (on/off) and intensity values
- Pack/unpack light data structures for save/load and network synchronization
- Convert legacy Marathon 1 light format to new format
- Hook into Lua scripting system for light activation events
- Support per-tag light control (toggle multiple lights by tag value)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `light_definition` | struct | Holds default light configuration including on/off sounds and static light data template |
| `lighting_function_specification` | struct | Parameters for one lighting animation state: function type, period, intensity range with variance |
| `light_data` | struct | Runtime light instance: current intensity, phase, period, state, and static configuration |
| `static_light_data` | struct | Immutable light config: type, flags, initial phase, 6 lighting specs (active/inactive/becoming states), tag |
| `old_light_data` | struct | Marathon 1 legacy format for backward compatibility (read during conversion) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LightList` | `vector<light_data>` | extern global | All active lights on current map; replaces fixed-size array |
| `light_definitions[NUMBER_OF_LIGHT_TYPES]` | array of `light_definition` | static global | 3 predefined light types: _normal_light, _strobe_light, _lava_light |
| `lighting_functions[NUMBER_OF_LIGHTING_FUNCTIONS]` | array of function pointers | static global | 4 animation functions: constant, linear, smooth, flicker |
| `rephase_light` | function | static (file scope) | Helper to advance light phase and trigger state transitions on overflow |
| `get_lighting_function_specification` | function | static (file scope) | Maps light state to its corresponding lighting_function_specification |

## Key Functions / Methods

### new_light
- Signature: `short new_light(struct static_light_data *data)`
- Purpose: Create a new light instance and add it to LightList; initialize all dynamic fields
- Inputs: Pointer to static_light_data configuration
- Outputs/Return: Light index (0ΓÇôMAXIMUM_LIGHTS_PER_MAPΓêÆ1) or NONE if full
- Side effects: Allocates slot in LightList; marks slot as used; initializes phase, state, intensity
- Calls: `SLOT_IS_FREE()`, `MARK_SLOT_AS_USED()`, `LIGHT_IS_INITIALLY_ACTIVE()`, `change_light_state()`, `lighting_function_dispatch()`, `get_lighting_function_specification()`, `rephase_light()`
- Notes: Idiot-proofs null input; initializes light in secondary state first, then transitions to primary state to set correct initial/final intensity values

### update_lights
- Signature: `void update_lights(void)`
- Purpose: Advance all active lights by one tick; recalculate intensity based on current phase
- Inputs: None (operates on global LightList)
- Outputs/Return: None; modifies light intensity in-place
- Side effects: Increments phase on each active light; calls rephase_light to handle state transitions
- Calls: `SLOT_IS_USED()`, `rephase_light()`, `lighting_function_dispatch()`, `get_lighting_function_specification()`
- Notes: Called once per game tick (1/30 second); frame-based animation

### change_light_state
- Signature: `void change_light_state(size_t light_index, short new_state)`
- Purpose: Transition light to new state (e.g., _light_becoming_active ΓåÆ _light_primary_active); set period, initial/final intensity with randomness
- Inputs: Light index, new state constant
- Outputs/Return: None; modifies light_data fields (phase, period, initial_intensity, final_intensity, state)
- Side effects: Resets phase to 0; picks random period delta and intensity delta within spec bounds; captures current intensity as initial_intensity
- Calls: `get_light_data()`, `get_lighting_function_specification()`, `global_random()`
- Notes: Provides randomness variance to avoid synchronized lighting; called by `rephase_light()` and `set_light_status()`

### rephase_light
- Signature: `static void rephase_light(short light_index)`
- Purpose: Check if light phase has overflowed period; if so, advance to next state and re-initialize
- Inputs: Light index
- Outputs/Return: None; modifies state and phase
- Side effects: Loops: subtracts period from phase, transitions state (e.g., _light_becoming_active ΓåÆ _light_primary_active), calls `change_light_state()` until phase < period
- Calls: `get_light_data()`, `change_light_state()`, `LIGHT_IS_STATELESS()`
- Notes: Stateless lights cycle through all 6 states; state-machine lights alternate between primary/secondary within active/inactive groups; invariant: final phase always < period

### set_light_status
- Signature: `bool set_light_status(size_t light_index, bool new_status)`
- Purpose: Toggle light on/off; only changes state if current status differs; triggers state machine and Lua hook
- Inputs: Light index, desired status (true = active, false = inactive)
- Outputs/Return: `true` if status changed, `false` otherwise
- Side effects: Calls `change_light_state()` to begin transition; calls `L_Call_Light_Activated()` Lua hook; updates switch panel state via `assume_correct_switch_position()`
- Calls: `get_light_data()`, `get_light_status()`, `LIGHT_IS_STATELESS()`, `change_light_state()`, `L_Call_Light_Activated()`, `assume_correct_switch_position()`
- Notes: Returns false for stateless lights (no state change possible); skips Lua hook if light is stateless

### set_tagged_light_statuses
- Signature: `bool set_tagged_light_statuses(short tag, bool new_status)`
- Purpose: Set on/off status for all lights sharing a specific tag value
- Inputs: Tag value (0 = no tag, 1ΓÇô32767 = match lights with that tag), desired status
- Outputs/Return: `true` if any light's status changed, `false` otherwise
- Side effects: Iterates all lights in LightList; calls `set_light_status()` on matches
- Calls: `set_light_status()`
- Notes: Combines results; supports groups of linked switches/lights

### lighting_function_dispatch
- Signature: `static _fixed lighting_function_dispatch(short function_index, _fixed initial_intensity, _fixed final_intensity, short phase, short period)`
- Purpose: Dispatch to correct lighting function (constant, linear, smooth, flicker) and return interpolated intensity
- Inputs: Function type (0ΓÇô3), initial/final intensity (fixed-point), phase, period
- Outputs/Return: Interpolated intensity at given phase
- Side effects: None
- Calls: Function pointer from `lighting_functions[]` array
- Notes: Asserts function_index in valid range; used by lighting animation loops

### Lighting animation functions
Four static procs compute intensity at a given phase:
- `constant_lighting_proc`: Returns `final_intensity` regardless of phase
- `linear_lighting_proc`: Linear interpolation: `initial + (final - initial) * phase / period`
- `smooth_lighting_proc`: Cosine-based smooth interpolation using `cosine_table[]`
- `flicker_lighting_proc`: Smooth curve with random jitter: `smooth + random_delta()`

### Serialization functions
- `unpack_old_light_data`, `pack_old_light_data`: Marathon 1 format (32 bytes)
- `unpack_static_light_data`, `pack_static_light_data`: Immutable light config (100 bytes)
- `unpack_light_data`, `pack_light_data`: Runtime light state including static data (128 bytes)
- All advance stream pointer and assert correct final size

### convert_old_light_data_to_new
- Signature: `void convert_old_light_data_to_new(static_light_data* NewLights, old_light_data* OldLights, int Count)`
- Purpose: Translate Marathon 1 light definitions to new format for backward compatibility
- Inputs: Count of old light entries to convert
- Outputs/Return: Populates NewLights array
- Side effects: Maps old light types to nearest new equivalent; copies and adjusts period/intensity; sets initial-active flag from old mode
- Calls: `get_defaults_for_light_type()`, `obj_copy()`, `SET_FLAG()`, `FixLightState()`
- Notes: Uses `FixLightState()` helper to rescale periods from TICKS_PER_SECOND units; ensures period ΓëÑ 1 to avoid rephase loop hang

## Control Flow Notes
Lights are initialized when lights are loaded from map data (called from map loading). Per-frame update occurs in `update_lights()`, which is called by the main game loop (inferred from architecture, not directly visible). State transitions occur on player interaction with light switches (via `set_light_status()` or `set_tagged_light_statuses()`), triggering Lua hooks. Lights persist across level boundaries in the global LightList until the map is unloaded.

## External Dependencies
- **Includes**: `cseries.h` (common definitions), `map.h` (map structure and constants), `lightsource.h` (header), `Packing.h` (serialization), `lua_script.h` (scripting hooks)
- **Symbols defined elsewhere**: 
  - `MAXIMUM_LIGHTS_PER_MAP`, `LIGHT_IS_INITIALLY_ACTIVE()`, `LIGHT_IS_STATELESS()` macros from lightsource.h
  - `GetMemberWithBounds()`, `SLOT_IS_USED()`, `MARK_SLOT_AS_USED()` macros (likely from map.h)
  - `global_random()` (random number generator)
  - `cosine_table[]`, `HALF_CIRCLE`, `TRIG_MAGNITUDE`, `TRIG_SHIFT` (trig lookup table)
  - `L_Call_Light_Activated()` (Lua scripting interface)
  - `assume_correct_switch_position()` (panel/switch synchronization)
  - `vhalt()`, `csprintf()`, `temporary` (error handling/formatting)
  - `StreamToValue()`, `ValueToStream()` functions from Packing.h
