# Source_Files/GameWorld/items.cpp

## File Purpose
Manages item lifecycle in the game world including spawning, pickup mechanics, inventory tracking, animation, and XML-based configuration of item properties. Handles both automatic and manual item collection, network-safe item placement, and environment-specific item availability.

## Core Responsibilities
- Create and spawn items on the map with network/difficulty context
- Implement item pickup logic with special handling for powerups, balls, weapons, and ammo
- Provide automatic nearby-item collection with geometry validation
- Trigger hidden items via zone-based flood-fill activation
- Update item animations and handle randomization of non-animated items
- Manage item definitions with runtime XML reconfiguration
- Track player inventory and item availability per environment/gamemode
- Lua scripting integration for item lifecycle events

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | (external) Item type attributes: shape, names, max count, invalid environments |
| `object_data` | struct | (external, from map.h) Item instance on map with location, animation state |
| `player_data` | struct | (external, from player.h) Player state including items[] inventory array |
| `XML_ItemParser` | class | Custom XML parser for item definition attributes (index, singular/plural names, max count, type) |
| `object_location` | struct | (external, from map.h) 3D position and polygon location for spawn |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `original_item_definitions` | `item_definition*` | static (module) | Backup copy of original item defs before XML modifications; used for ResetValues() |
| `ItemParser` | `XML_ItemParser` | static (module) | Parser instance for individual `<item>` tags |
| `ItemsParser` | `XML_ElementParser` | static (module) | Root parser for `<items>` container tag |
| `item_definitions` | `item_definition[]` | external (item_definitions.h) | Global array of item type definitions |

## Key Functions / Methods

### new_item
- **Signature:** `short new_item(struct object_location *location, short type)`
- **Purpose:** Spawn a new item instance on the map at the given location and type.
- **Inputs:** location (3D position + polygon), type (item type index)
- **Outputs/Return:** object_index (NONE if creation failed)
- **Side effects:** Creates `object_data` entry; updates object owner to `_object_is_item`; marks item for placement tracking; calls Lua hook `L_Call_Item_Created`; may make item invisible if network-only in single-player
- **Calls:** `new_map_object()`, `get_item_definition()`, `get_item_kind()`, `object_was_just_added()`, `L_Call_Item_Created()`
- **Notes:** Filters items based on `invalid_environments` flags and `current_game_has_balls()`. Network-only items are spawned invisible; permutation set to NONE for network-only items in single-player. Assertion checks `NUMBER_OF_DEFINED_ITEMS` consistency.

### try_and_add_player_item
- **Signature:** `bool try_and_add_player_item(short player_index, short type)`
- **Purpose:** Attempt to add item to player's inventory, handling powerups/balls/weapons/ammo with special logic per item kind.
- **Inputs:** player_index, item type
- **Outputs/Return:** success (bool)
- **Side effects:** Modifies `player->items[type]` array; calls `process_player_powerup()` for powerups; calls `process_new_item_for_reloading()` for weapons/ammo; calls `mark_player_inventory_as_dirty()`; calls Lua hook `L_Call_Got_Item()`; plays pickup sound and screen flash if local player
- **Calls:** `legal_player_powerup()`, `process_player_powerup()`, `object_was_just_destroyed()`, `find_player_ball_color()`, `process_new_item_for_reloading()`, `mark_player_inventory_as_dirty()`, `L_Call_Got_Item()`, `SoundManager::instance()->PlayLocalSound()`, `start_fade()`
- **Notes:** Powerups are consumed immediately (not stored); balls only one per player with team-based logic (CTF); weapons/ammo respect `maximum_count_per_player` except ammo on total_carnage level. Handles both first pickup and increment cases.

### swipe_nearby_items
- **Signature:** `void swipe_nearby_items(short player_index)`
- **Purpose:** Auto-collect all reachable items in adjacent polygons within arm's reach with LOS checks.
- **Inputs:** player_index
- **Outputs/Return:** (void)
- **Side effects:** Removes collected items from map; modifies player inventory via `get_item()`; restarts search when item collected (to handle dynamic list changes)
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_polygon_data()`, `get_map_indexes()`, `guess_distance2d()`, `get_monster_dimensions()`, `test_item_retrieval()`, `get_item()`
- **Notes:** MAXIMUM_ARM_REACH = 3*WORLD_ONE_FOURTH. Searches current polygon plus all neighbors via adjacent polygon indexes. Line-of-sight validated via `test_item_retrieval()`. Z-coordinate bounds checked against player height. Restarting search after each collection prevents iterator invalidation.

### trigger_nearby_items
- **Signature:** `void trigger_nearby_items(short polygon_index)`
- **Purpose:** Activate hidden (invisible) items in reachable zones via flood-fill from entry polygon, teleporting them into world.
- **Inputs:** polygon_index (starting zone)
- **Outputs/Return:** (void)
- **Side effects:** Makes invisible items visible via `teleport_object_in()`; clears OBJECT_IS_INVISIBLE flag and permutation
- **Calls:** `flood_map()`, `get_polygon_data()`, `get_object_data()`, `teleport_object_in()`
- **Notes:** Uses `item_trigger_cost_function()` to traverse zones (zone borders have cost -1 = stop). Iterative flood-fill pattern; NONE polygon ends iteration.

### animate_items
- **Signature:** `void animate_items(void)`
- **Purpose:** Update animation state of all visible item objects each frame, handling both animated shapes and non-animated shape randomization.
- **Inputs:** (none; operates on global objects list)
- **Outputs/Return:** (void)
- **Side effects:** Updates `object->facing` field; calls `randomize_object_sequence()` once per non-animated item; calls `animate_object()` for animated shapes
- **Calls:** `get_item_definition()`, `get_shape_animation_data()`, `randomize_object_sequence()`, `animate_object()`
- **Notes:** Uses `object->facing >= 0` as flag for "not yet randomized"; sets to NONE after randomization. Non-animated items randomized only once on first frame encountered. Skips invisible items.

### get_item
- **Signature:** `static bool get_item(short player_index, short object_index)`
- **Purpose:** Core item pickup implementation: validate and add item to player, then remove from map.
- **Inputs:** player_index, object_index
- **Outputs/Return:** success (bool)
- **Side effects:** Calls `try_and_add_player_item()`; removes object from map if successful via `remove_map_object()`
- **Calls:** `get_object_data()`, `try_and_add_player_item()`, `remove_map_object()`
- **Notes:** Assertion validates object is `_object_is_item`. Wraps `try_and_add_player_item()` with cleanup.

### test_item_retrieval
- **Signature:** `static bool test_item_retrieval(short polygon_index1, world_point3d *location1, world_point3d *location2)`
- **Purpose:** Validate line-of-sight from player to item, respecting walls, solid lines, and moving platforms.
- **Inputs:** starting polygon, player location, item location
- **Outputs/Return:** valid_retrieval (bool, true if unobstructed)
- **Side effects:** (none)
- **Calls:** `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `get_polygon_data()`, `get_platform_data()`
- **Notes:** Ray-walks from player to item location checking each line crossed. Returns false if solid line encountered or moving platform blocking path. Stops iteration on polygon==NONE or valid_retrieval==false.

### get_item_definition
- **Signature:** `item_definition *get_item_definition(const short type)`
- **Purpose:** Safe accessor for item definition array with bounds checking.
- **Inputs:** item type index
- **Outputs/Return:** pointer to item_definition (or NULL on out-of-bounds)
- **Side effects:** (none)
- **Calls:** `GetMemberWithBounds()`
- **Notes:** Inlined for external use; `get_item_definition_external()` provides non-inlined version for external callers.

## Control Flow Notes
**Initialization:** `initialize_items()` is a no-op stub (animations handled per-frame).

**Per-frame updates:** `animate_items()` runs every frame, updating all item animations and randomizing shapes on first encounter.

**Item spawn:** `new_item()` called during level load by placement system; filters by network/single-player mode and environment flags.

**Item collection:** Two pathsΓÇöautomatic via `swipe_nearby_items()` (per player update) or manual via `get_item()` when triggered. Both converge on `try_and_add_player_item()` for inventory logic.

**Zone activation:** `trigger_nearby_items()` called when player enters item-trigger polygon; uses flood-fill to activate all hidden items in connected zones.

**Configuration:** XML parser allows runtime modification of item definitions; original state backed up and can be reset.

## External Dependencies
- **map.h:** `object_data`, `object_location`, `polygon_data`, `world_point3d`, `world_point2d`, `new_map_object()`, `remove_map_object()`, `get_object_data()`, `get_polygon_data()`, `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `teleport_object_in()`, `randomize_object_sequence()`, `animate_object()`
- **player.h:** `player_data`, `get_player_data()`, `process_new_item_for_reloading()`, `mark_player_inventory_as_dirty()`, `legal_player_powerup()`, `process_player_powerup()`
- **monsters.h:** `get_monster_data()`, `get_monster_dimensions()`
- **interface.h:** `get_shape_animation_data()`, `get_item_kind()` (defined elsewhere)
- **SoundManager.h:** `SoundManager::instance()->PlayLocalSound()`, sound index accessors
- **platforms.h:** `get_platform_data()`, `PLATFORM_IS_MOVING()`
- **fades.h:** `start_fade()`
- **lua_script.h:** `L_Call_Item_Created()`, `L_Call_Got_Item()`
- **network_games.h:** `current_game_has_balls()`, `dynamic_world->player_count`, game type/environment flags
- **flood_map.h:** `flood_map()`
- **item_definitions.h:** `item_definitions[]` array, `NUMBER_OF_DEFINED_ITEMS`
