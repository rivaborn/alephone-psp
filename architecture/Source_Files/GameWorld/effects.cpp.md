# Source_Files/GameWorld/effects.cpp

## File Purpose
Manages temporary visual and audio effects in the game worldΓÇöexplosions, teleport effects, blood splashes, sparks, and other short-lived phenomena. Effects are tied to animated map objects and are automatically removed when their animations complete or after a delay period expires.

## Core Responsibilities
- Create new effects at world locations with specified types and animations
- Update all active effects each frame, advancing animations and delay timers
- Remove effects when animations terminate or all nonpersistent effects are cleared
- Manage effect-definition lookups and validation
- Handle specialized teleport effects (fold in/fold out)
- Mark effect collections for loading/unloading as needed
- Serialize and deserialize effect data for save games and network play

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `effect_data` | struct | Active effect instance: type, associated object, flags, generic data field, delay counter, unused padding. 32 bytes. |
| `effect_definition` | struct | Defines an effect template: collection/shape, sound pitch, flags, optional delay, delay sound. 14 bytes. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effects` | `effect_data*` | global (macro) | Array of active effects, points to EffectList[0]; size bound by MAXIMUM_EFFECTS_PER_MAP. |
| `effect_definitions` | `effect_definition[]` | static | Working copy of effect definitions, initialized from `original_effect_definitions`. |

## Key Functions / Methods

### new_effect
- Signature: `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- Purpose: Create a new effect at a world location. If sound-only, plays sound and returns NONE. Otherwise, allocates a free effect slot, creates an associated map object, initializes animation state.
- Inputs: 3D position, containing polygon, effect type enum, facing angle
- Outputs/Return: Effect index on success, NONE on failure
- Side effects: Allocates new map object; marks effect slot as used; may call sound playback
- Calls: `get_effect_definition()`, `get_shape_animation_data()`, `play_world_sound()`, `new_map_object3d()`, `get_object_data()`, `SET_OBJECT_OWNER()`, `SET_OBJECT_INVISIBILITY()`
- Notes: Validates polygon_index and definition before proceeding. Handles optional delay (effect initially invisible). Sound-only effects do not consume an effect slot.

### update_effects
- Signature: `void update_effects(void)`
- Purpose: Advance all active effects by one tick. Decrement delay counters, animate visible effects, detect animation completion, remove finished effects, optionally make twin objects visible.
- Inputs: None (reads global effects array)
- Outputs/Return: None
- Side effects: Modifies effect delay/visibility; calls `animate_object()`, `SET_OBJECT_INVISIBILITY()`, `play_object_sound()`, `remove_effect()`
- Calls: `get_object_data()`, `get_effect_definition()`, `GET_OBJECT_ANIMATION_FLAGS()`, `remove_effect()`
- Notes: Loops all effect slots; continues on null definition (idiot-proofing). Checks flags `_end_when_animation_loops` and `_end_when_transfer_animation_loops` to determine when to remove.

### remove_effect
- Signature: `void remove_effect(short effect_index)`
- Purpose: Deactivate an effect, remove its associated object from the world.
- Inputs: Effect index
- Outputs/Return: None
- Side effects: Marks effect slot as free; removes map object
- Calls: `get_effect_data()`, `remove_map_object()`, `MARK_SLOT_AS_FREE()`

### remove_all_nonpersistent_effects
- Signature: `void remove_all_nonpersistent_effects(void)`
- Purpose: Sweep all effect slots and remove any that end when animations loop (used during level changes).
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `remove_effect()` on matching effects
- Calls: `get_effect_definition()`, `remove_effect()`

### teleport_object_out / teleport_object_in
- Signature: `void teleport_object_out(short object_index)` / `void teleport_object_in(short object_index)`
- Purpose: Create specialized teleport fold-out or fold-in effects. Out-effect makes object invisible; in-effect restores visibility.
- Inputs: Object index to teleport
- Outputs/Return: None
- Side effects: Modifies object visibility, shape, transfer mode; stores object_index in effect.data field for later retrieval
- Calls: `get_object_data()`, `new_effect()`, `get_effect_data()`, `play_object_sound()`, `Sound_TeleportOut()`
- Notes: In-effect checks for existing in-effect to avoid duplicates. Copies object shape/flags to effect object for visual match.

### get_effect_data / get_effect_definition
- Signature: `effect_data *get_effect_data(short effect_index)` / `effect_definition *get_effect_definition(short type)`
- Purpose: Safe accessor functions. Validate bounds and slot usage, assert/return on failure.
- Inputs: Index or type
- Outputs/Return: Pointer to effect_data or effect_definition, or NULL (definition only)
- Side effects: None
- Calls: `GetMemberWithBounds()`, `vassert()`

### Serialization functions
- `unpack_effect_data()` / `pack_effect_data()` ΓÇô convert between stream and effect_data array
- `unpack_effect_definition()` / `pack_effect_definition()` ΓÇô convert between stream and effect_definition array
- All advance stream pointer and assert size matches; no side effects beyond pointer mutation.

### init_effect_definitions
- Signature: `void init_effect_definitions(void)`
- Purpose: Reset effect definitions to hardcoded defaults via memcpy.
- Inputs: None
- Outputs/Return: None
- Side effects: Overwrites static `effect_definitions[]` array

## Control Flow Notes
Within the frame/update cycle:
1. **Creation**: Gameplay code calls `new_effect()` or `teleport_object_out()/in()` in response to game events (projectile impact, monster death, item pickup, teleportation).
2. **Update**: Main loop calls `update_effects()` each tick to animate and decay effects.
3. **Cleanup**: `remove_all_nonpersistent_effects()` called during level transitions; individual `remove_effect()` called when animations finish.
4. **Serialization**: Packing functions used when saving/loading maps or transmitting over network.

## External Dependencies
- **Notable includes**: `cseries.h` (common types/macros), `map.h` (world structures, object accessors), `interface.h` (shape/sound data), `effects.h` (effect declarations), `SoundManager.h` (sound playback), `Packing.h` (serialization macros), `effect_definitions.h` (hardcoded effect table).
- **Defined elsewhere**: `effects` array (EffectList global in map.cpp), `effect_definitions` template (effect_definitions.h), `new_map_object3d()`, `animate_object()`, `play_object_sound()`, `remove_map_object()`, `get_shape_animation_data()` (map/objects/shapes modules).
