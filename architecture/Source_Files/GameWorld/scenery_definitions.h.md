# Source_Files/GameWorld/scenery_definitions.h

## File Purpose
Defines the properties, appearance, and behavior of static scenery objects (lamps, debris, containers, etc.) in the game world. Contains a static array of 61 scenery definitions organized by environment theme (lava, water, sewage, alien, Jjaro), each with collision radius, height, destruction effects, and state flags.

## Core Responsibilities
- Define scenery behavior flags (`_scenery_is_solid`, `_scenery_is_animated`, `_scenery_can_be_destroyed`)
- Declare the `scenery_definition` struct to store per-object metadata
- Populate static `scenery_definitions[]` array with 61 environment objects
- Provide lookup constant `NUMBER_OF_SCENERY_DEFINITIONS` for bounds
- Associate destruction effects (lamp breakage, explosions) with destroyable scenery

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `scenery_definition` | struct | Stores flags, shape reference, collision bounds (radius/height), destruction effect ID, and destroyed shape |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scenery_definitions` | array of `scenery_definition` | static | Array of 61 predefined scenery objects indexed by ID |
| `_scenery_is_solid` | enum constant (0x0001) | static | Flag: object blocks movement/projectiles |
| `_scenery_is_animated` | enum constant (0x0002) | static | Flag: object is animated (marked unused; sequences define animation) |
| `_scenery_can_be_destroyed` | enum constant (0x0004) | static | Flag: object can be destroyed, triggering effect and shape change |

## Key Functions / Methods
NoneΓÇöthis is a data definition file.

## Control Flow Notes
**Initialization phase**: Included during compilation to populate the game's scenery object database. The array is indexed by scenery type ID during level loading to retrieve properties. **Gameplay**: Collision system queries `radius`/`height` for bounds checking; destruction system invokes `destroyed_effect` and swaps to `destroyed_shape` when scenery is damaged.

## External Dependencies
- **Macros (defined elsewhere)**: `BUILD_DESCRIPTOR()` ΓÇö constructs shape references
- **Type definitions (defined elsewhere)**: `shape_descriptor`, `world_distance`, `uint16`, `int16`
- **Constants (defined elsewhere)**: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `NONE`
- **Effect symbols (defined elsewhere)**: `_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, `_effect_sewage_lamp_breaking`, `_effect_alien_lamp_breaking`, `_effect_grenade_explosion`
- **Collection symbols (defined elsewhere)**: `_collection_scenery1`, `_collection_scenery2`, `_collection_scenery3`, `_collection_scenery4`, `_collection_scenery5` ΓÇö texture/sprite atlases
