# Source_Files/GameWorld/placement.cpp

## File Purpose
Manages the dynamic placement and respawning of monsters and items in the game world. Loads placement configuration from map files, places initial objects, periodically recreates objects based on difficulty/gameplay rules, and provides safe player spawn locations.

## Core Responsibilities
- Load placement frequency data (initial/min/max/random counts) for all monster and item types from map WAD
- Place initial objects at game start according to placement rules and predefined map locations
- Periodically recreate monsters and items to maintain minimum counts and random encounters
- Track current object counts and trigger replacement when objects are destroyed
- Validate polygon locations for safe object placement (avoid walls, other objects, players)
- Select random player spawn locations away from monsters and other players
- Manage asset loading (collections and sounds) for monsters that may be placed

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `object_frequency_definition` | struct (from map.h) | Defines spawn frequency, min/max counts, and flags for each object type |
| `object_location` | struct (from map.h) | Position (3D), polygon index, facing angle, and flags for placed object |
| `map_object` | struct (from map.h) | Saved predefined object location from map editor (monsters, items, scenery) |
| `polygon_data` | struct (from map.h) | Polygon geometry; used to validate placement legality |
| `object_data` | struct (from map.h) | Runtime game object (location, owner, shape, animation state) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `object_placement_info` | `object_frequency_definition[2*MAXIMUM_OBJECT_TYPES]` | static | Combined array storing both monster and item placement rules; split into two halves |
| `monster_placement_info` | `object_frequency_definition*` | static | Pointer into `object_placement_info` pointing to monster half |
| `item_placement_info` | `object_frequency_definition*` | static | Pointer into `object_placement_info` pointing to item half |

## Key Functions / Methods

### load_placement_data
- **Signature:** `void load_placement_data(uint8 *_monsters, uint8 *_items)`
- **Purpose:** Initialize placement rules from map data stream. Unpacks binary placement data and sets up monster/item frequency arrays. Called once per map load.
- **Inputs:** Byte streams for monster and item placement data (from WAD)
- **Outputs/Return:** None (modifies global `object_placement_info`)
- **Side effects:** Clears and populates global placement info arrays. Clears marine monster type (hardcoded as player-only). Debug-mode validation of placement counts.
- **Calls:** `objlist_clear`, `unpack_object_frequency_definition`, `obj_clear`, `dprintf`
- **Notes:** Marine (monster #0) always has placement cleared to prevent spawning as enemy. #if 0 block contains disabled network-only item fixup logic.

### place_initial_objects
- **Signature:** `void place_initial_objects(void)`
- **Purpose:** Spawn all initial monsters and items at game start according to placement rules and predefined map locations.
- **Inputs:** None (reads from global `monster_placement_info`, `item_placement_info`, `dynamic_world`)
- **Outputs/Return:** None (creates monsters/items via engine)
- **Side effects:** Calls `new_monster()` and `new_item()` to create objects. Updates `dynamic_world->random_monsters_left[]` and `random_items_left[]` counters.
- **Calls:** `add_objects` (twice: monsters, items)
- **Notes:** Monsters only placed if `_monsters_replenish` game option is set. Civilian casualty counters zeroed at start.

### recreate_objects
- **Signature:** `void recreate_objects(void)`
- **Purpose:** Periodic check (roughly every 0.5 seconds) to respawn monsters and items if counts fall below configured minimums or to add random encounters.
- **Inputs:** None (reads `dynamic_world->tick_count`)
- **Outputs/Return:** None
- **Side effects:** Calls `_recreate_objects` to spawn new objects. Updates delay timer.
- **Calls:** `_recreate_objects` (twice: monsters, items)
- **Notes:** Delay timer resets if game time goes backwards (new game started). Only recreates monsters if `_monsters_replenish` game option is set.

### object_was_just_added
- **Signature:** `void object_was_just_added(short object_class, short object_type)`
- **Purpose:** Callback invoked when a new monster or item is created. Updates object count tracking.
- **Inputs:** Object class (`_object_is_monster` or `_object_is_item`), object type index
- **Outputs/Return:** None
- **Side effects:** Increments `dynamic_world->current_monster_count[]` or `current_item_count[]`
- **Calls:** None (simple increment)
- **Notes:** Called by `new_monster()` and `new_item()` to keep counts synchronized.

### object_was_just_destroyed
- **Signature:** `void object_was_just_destroyed(short object_class, short object_type)`
- **Purpose:** Callback when a monster or item is destroyed. Decrements count and may spawn a replacement if below minimum.
- **Inputs:** Object class, object type index
- **Outputs/Return:** None
- **Side effects:** Decrements count. Calls `add_objects(ΓÇª, 1, false)` to respawn if count fell below minimum.
- **Calls:** `add_objects` (conditional)
- **Notes:** For items, handles case where player held items not initially placed. Respawn is non-initial, so may use random location if configured.

### get_random_player_starting_location_and_facing
- **Signature:** `short get_random_player_starting_location_and_facing(short max_player_index, short team, struct object_location *location)`
- **Purpose:** Find a safe spawn location for a player on the current map. Picks from predefined starts but avoids monsters and other players.
- **Inputs:** `max_player_index` (how many players already spawned), `team` (team filter, or NONE for any), pointer to output location struct
- **Outputs/Return:** Starting location index; outputs filled `location` struct (position, polygon, facing)
- **Side effects:** Modifies output location struct
- **Calls:** `get_player_starting_location_and_facing()` (from game_wad.c), `point_is_player_visible()`, `point_is_monster_visible()`
- **Notes:** Picks offset start and tries all defined starts. Ranks by combined distance to nearest player and monster (player distance weighted heavier). Falls back to last candidate if all starts occupied.

---

## Control Flow Notes
**Initialization chain:**
1. Map loads ΓåÆ `load_placement_data()` unpacks placement rules
2. Game starts ΓåÆ `mark_all_monster_collections()` and `load_all_monster_sounds()` load assets
3. Level initializes ΓåÆ `place_initial_objects()` spawns all initial monsters/items
4. Players spawn ΓåÆ `get_random_player_starting_location_and_facing()` picks safe starts

**Runtime loop:**
- Every frame: `update_world()` (marathon.c) calls `recreate_objects()`
- On object creation: object factory calls `object_was_just_added()`
- On object destruction: engine calls `object_was_just_destroyed()` (may trigger respawn)

## External Dependencies
**Notable includes / imports:**
- `cseries.h` ΓÇö core utilities, debug macros (`dprintf`, `assert`)
- `map.h` ΓÇö global world data (`dynamic_world`, `saved_objects`), geometry accessors, placement structs
- `monsters.h` ΓÇö `new_monster()`, `activate_monster()`, `find_closest_appropriate_target()`
- `items.h` ΓÇö `new_item()`
- `<string.h>` ΓÇö `string` functions (minimal use)

**Key external symbols (defined elsewhere):**
- `dynamic_world` ΓÇö global dynamic game state (object counts, tick counter)
- `saved_objects` ΓÇö map-editor-placed object locations
- `new_monster()`, `new_item()` ΓÇö object factories
- `activate_monster()`, `find_closest_appropriate_target()` ΓÇö monster behavior
- `global_random()` ΓÇö RNG
- `point_is_player_visible()`, `point_is_monster_visible()` ΓÇö visibility checks
- `get_polygon_data()`, `get_object_data()` ΓÇö map data accessors
- `GET_GAME_OPTIONS()` ΓÇö game settings macro
- Polygon/geometry utilities: `find_center_of_polygon()`, `find_new_object_polygon()`, `get_player_starting_location_and_facing()`
