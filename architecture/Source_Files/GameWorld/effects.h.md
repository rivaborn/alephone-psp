# Source_Files/GameWorld/effects.h

## File Purpose
Header file for managing visual effects (explosions, blood splashes, water effects, etc.) in the Marathon/Aleph One game engine. Defines effect data structures, enumeration of all effect types, and function prototypes for creating, updating, and serializing effects.

## Core Responsibilities
- Define the `effect_data` structure to store active effect instances
- Enumerate all 80+ effect types (explosions, ricochets, splashes, sparks, detonations)
- Maintain a dynamic array (`EffectList`) of active effects in the current game world
- Provide creation and lifecycle management (create, update, remove effects)
- Support serialization/deserialization of effect data and definitions
- Retrieve and manage effect definitions (loaded from resources)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `effect_data` | struct | Instance of an active effect; contains type, object reference, flags, timing, and metadata (16 bytes, padded to 32) |
| Effect type enum | enum | 80+ named constants (`_effect_rocket_explosion`, `_effect_grenade_contrail`, etc.) identifying effect categories |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EffectList` | `vector<effect_data>` | global | Dynamic array of all active effects in the current map |
| `effects` | macro | global | Backward-compatibility alias to `&EffectList[0]` for C-style array access |
| `MAXIMUM_EFFECTS_PER_MAP` | macro | global | Dynamic limit on active effects, retrieved via `get_dynamic_limit(_dynamic_limit_effects)` |

## Key Functions / Methods

### new_effect
- Signature: `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- Purpose: Create a new active effect at a specified world location
- Inputs: World position, polygon, effect type, facing angle
- Outputs/Return: Effect index (short) for later reference
- Side effects: Allocates and inserts into `EffectList`
- Calls: Not visible in header
- Notes: Subject to `MAXIMUM_EFFECTS_PER_MAP` limit

### update_effects
- Signature: `void update_effects(void)`
- Purpose: Advance all active effects by one game tick
- Inputs: None (operates on global `EffectList`)
- Outputs/Return: None
- Side effects: Modifies effect state, removes expired effects
- Calls: Not visible in header
- Notes: Assumes fixed ╬öt = 1 tick; called once per frame

### remove_effect
- Signature: `void remove_effect(short effect_index)`
- Purpose: Immediately destroy a specific effect instance
- Inputs: Effect index
- Outputs/Return: None
- Side effects: Marks slot as free (slot macros used: `MARK_SLOT_AS_FREE()`)
- Calls: Not visible in header

### remove_all_nonpersistent_effects
- Signature: `void remove_all_nonpersistent_effects(void)`
- Purpose: Clear all temporary effects (e.g., when unloading a level)
- Inputs: None
- Outputs/Return: None
- Side effects: Clears `EffectList` of non-persistent effects

### get_effect_data
- Signature: `effect_data *get_effect_data(const short effect_index)`
- Purpose: Accessor to retrieve effect data by index
- Inputs: Effect index
- Outputs/Return: Pointer to `effect_data` struct
- Side effects: None (read-only accessor)

### teleport_object_in / teleport_object_out
- Signature: `void teleport_object_in(short object_index)`, `void teleport_object_out(short object_index)`
- Purpose: Trigger teleportation effects for an object entering or exiting a portal
- Inputs: Object index
- Outputs/Return: None
- Side effects: Creates new `_effect_teleport_object_in` or `_effect_teleport_object_out` effect

### Serialization functions
- `unpack_effect_data`, `pack_effect_data`: Serialize/deserialize active effect instances
- `unpack_effect_definition`, `pack_effect_definition`: Serialize/deserialize effect definitions
- `init_effect_definitions`: Load effect definitions (source/format not visible)
- Signature: `uint8 *pack_effect_data(uint8 *Stream, effect_data *Objects, size_t Count)` (and unpack variant)
- Purpose: Convert effect data to/from binary for save files or network
- Inputs: Stream pointer, object array, count
- Outputs/Return: Updated stream pointer (for chaining)

## Control Flow Notes
Effects integrate into the game's main update loop: `new_effect()` creates instances (often triggered by weapons, damage, or environmental events), `update_effects()` is called once per frame to age all active effects, and `remove_effect()` / `remove_all_nonpersistent_effects()` clean them up when expired or on unload. The `delay` field allows effects to remain inactive for a configurable number of ticks before becoming visible.

## External Dependencies
- `dynamic_limits.h`: Provides `get_dynamic_limit()` to retrieve dynamic effect cap
- External types: `world_point3d`, `angle` (defined elsewhere)
- C++ STL: `vector<>` for effect list management
- Slot management macros: `SLOT_IS_USED()`, `SLOT_IS_FREE()`, `MARK_SLOT_AS_FREE()`, `MARK_SLOT_AS_USED()` (defined elsewhere)
