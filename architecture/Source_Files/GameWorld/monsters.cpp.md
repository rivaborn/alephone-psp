# Source_Files/GameWorld/monsters.cpp

## File Purpose
Implements the monster/NPC system for the game engine, including lifecycle management, AI behavior, pathfinding, combat, and serialization. Handles all non-player agents in the game world.

## Core Responsibilities
- **Monster lifecycle**: creation, activation, deactivation, and death
- **AI & targeting**: target selection, line-of-sight checks, mode/action transitions
- **Pathfinding**: cost-function-based path generation and navigation
- **Combat**: attack execution, damage calculation, projectile positioning
- **Physics & animation**: vertical/horizontal movement, collision detection, animation sequencing
- **Environmental interaction**: terrain features, platforms, media (liquid/hazards), doors
- **Serialization**: binary packing/unpacking of monster state and definitions
- **Configuration**: XML parsing for damage-kick parameters

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| monster_data | struct | Per-instance runtime state: position, vitality, target, animation, pathfinding |
| monster_definition | struct | Static per-type definition: model, sounds, AI params, attacks, physics |
| monster_pathfinding_data | struct | Transient data passed to cost function during pathfinding |
| damage_kick_definition | struct | Configures how much external velocity damage imparts (base + multiplier) |
| attack_definition | struct | Defines melee/ranged attack: type, repetitions, error, range, origin offsets |
| XML_DamageKickParser | class | Parses damage-kick XML elements and modifies damage_kick_definitions[] |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| monsters | vector\<monster_data\> | extern (map.h) | All active monsters on map |
| monster_definitions | monster_definition[] | extern (monster_definitions.h) | Definitions for all monster types |
| IntersectedObjects | vector\<short\> | static | Growable buffer for collision checks (reused each call) |
| damage_kick_definitions | damage_kick_definition[] | global | Damage-to-velocity mappings per damage type |
| original_damage_kick_definitions | damage_kick_definition* | static | Backup for XML reset functionality |
| DamageKickParser | XML_DamageKickParser | static | XML element parser for damage kicks |
| dynamic_world | dynamic_data* | extern (map.h) | Pointer to game state (tick count, last indices, etc.) |

## Key Functions / Methods

### new_monster
- **Signature:** `short new_monster(object_location *location, short monster_type)`
- **Purpose:** Create and place a new monster on the map at startup
- **Inputs:** spawn location, monster type (index into monster_definitions)
- **Outputs/Return:** monster index (allocated from monsters array) or NONE on failure
- **Side effects:** allocates monster slot, creates object_data, calls object_was_just_added(), may be dropped/promoted/demoted based on difficulty
- **Calls:** get_monster_definition(), new_map_object(), get_object_data(), nearest_goal_polygon_index(), object_was_just_added()
- **Notes:** Difficulty scaling: lower difficulties drop/demote aliens; higher difficulties promote minors. Blind/deaf/float flags copied from map_object. Vitality set to NONE triggers lazy initialization on first activation.

### move_monsters
- **Signature:** `void move_monsters(void)`
- **Purpose:** Main per-tick update loop for all active monsters; orchestrates animation, targeting, pathfinding, attacks
- **Inputs:** none (uses globals: monsters, dynamic_world)
- **Outputs/Return:** none
- **Side effects:** updates monster state, object animation, external/vertical velocity, modes, paths; may deactivate/kill monsters; resets last_monster_index_to_get_time
- **Calls:** get_object_data(), get_monster_definition(), cause_polygon_damage(), SET_MONSTER_IDLE_STATUS(), update_monster_vertical_physics_model(), animate_object(), change_monster_target(), find_closest_appropriate_target(), generate_new_path_for_monster(), handle_moving_or_stationary_monster(), execute_monster_attack(), try_monster_attack(), set_monster_action(), monster_needs_path(), clear_line_of_sight(), update_monster_physics_model(), cause_shrapnel_damage(), kill_monster(), teleport_object_out(), remove_map_object(), L_Invalidate_Monster()
- **Notes:** Paces target searches and path building across frames using last_monster_index_to_get_time/last_monster_index_to_build_path. Inactive monsters still scan for targets once per frame. Civilian-kill counter decremented periodically.

### monster_died
- **Signature:** `void monster_died(short target_index)`
- **Purpose:** Cleanup when a monster dies; orphan projectiles, retarget monsters that were locked on it
- **Inputs:** index of the dying monster
- **Outputs/Return:** none
- **Side effects:** orphans projectiles, marks dying monster unlocked, deletes its path; all monsters locked on target switch to find_closest_appropriate_target()
- **Calls:** get_monster_data(), orphan_projectiles(), set_monster_mode(), delete_path(), SET_MONSTER_NEEDS_PATH_STATUS(), find_closest_appropriate_target(), play_object_sound(), monster_needs_path(), change_monster_target(), set_monster_action()
- **Notes:** Called before target is expunged from array. Player monsters don't orphan projectiles (correct damage attribution).

### monster_pathfinding_cost_function
- **Signature:** `long monster_pathfinding_cost_function(short src_poly, short line_idx, short dst_poly, void *data)`
- **Purpose:** Cost function for A*/breadth-first pathfinding; evaluates traversability and preference of moving from src to dst
- **Inputs:** source polygon, line between them, destination polygon, pathfinding_data (contains definition, cross_zone_boundaries flag)
- **Outputs/Return:** cost (long) or -1 (impassable)
- **Side effects:** none
- **Calls:** get_polygon_data(), get_line_data(), LINE_IS_SOLID(), LINE_IS_VARIABLE_ELEVATION(), monster_can_enter_platform(), monster_can_leave_platform()
- **Notes:** Returns -1 for solid lines, platform access violations, height mismatches, zone borders (if not crossing), too-narrow passages, dangerous media. Adds area+platform/obstruction penalties. Prefers lower height changes. Media boundaries cost extra.

### find_obstructing_terrain_feature
- **Signature:** `short find_obstructing_terrain_feature(short monster_idx, short *feature_idx, short *relevant_poly_idx)`
- **Purpose:** Detect platforms/doors/height changes in monster's path; allows standing on ledges or waiting for platform
- **Inputs:** monster index; output pointers for feature type index and relevant polygon
- **Outputs/Return:** feature type (_standing_on_sniper_ledge, _entering_platform_polygon, _leaving_platform_polygon, _flying_or_floating_transition) or NONE
- **Side effects:** may update monster->desired_height; may call monster_needs_path()
- **Calls:** get_monster_data(), get_monster_definition(), get_object_data(), ray_to_line_segment(), find_line_crossed_leaving_polygon(), get_polygon_data(), find_adjacent_polygon(), get_media_data(), IsMediaDangerous()
- **Notes:** Traces ray forward from monster in facing direction. Handles floating/flying monsters differently (can adjust height). Sniper ledges used if monster is locked and floor is >minimum height above desired. Media-wading logic: dangerous media ΓåÆ no wading, otherwise wade to half-monster height.

### position_monster_projectile
- **Signature:** `short position_monster_projectile(short aggressor_idx, short target_idx, attack_definition *attack, world_point3d *origin, world_point3d *dst, world_point3d *vec, angle theta)`
- **Purpose:** Calculate projectile spawn point and direction for monster attack
- **Inputs:** attacker/target monster indices, attack definition (offsets), facing angle theta
- **Outputs/Return:** polygon containing origin point; fills origin, destination, vector, updates aggressor->elevation
- **Side effects:** modifies elevation of aggressor monster
- **Calls:** get_monster_data(), get_object_data(), get_monster_dimensions(), find_new_object_polygon(), isqrt()
- **Notes:** If destination is NULL, fires along monster's facing at aggressor's elevation; otherwise shoots toward target 3/4 up its body. Offsets applied in order: dz (vertical), dy (perpendicular), dx (forward).

### SetPlayerViewAttribs
- **Signature:** `void SetPlayerViewAttribs(int16 half_visual_arc, int16 half_vertical_visual_arc, world_distance visual_range, world_distance dark_visual_range)`
- **Purpose:** Modify player-as-monster visual attributes when aiming guided projectiles
- **Inputs:** visual arc angles, light/dark ranges (or 0/NONE for no change)
- **Outputs/Return:** none
- **Side effects:** modifies monster_definitions[_monster_marine] (player's monster entry)
- **Calls:** none
- **Notes:** Only updates field if new value > 0, preserving original if not specified.

### unpack_monster_data / pack_monster_data
- **Signature:** `uint8 *unpack_monster_data(uint8 *Stream, monster_data *Objects, size_t Count)` and vice versa
- **Purpose:** Serialize/deserialize monster runtime state to/from byte streams (savegames, network)
- **Inputs:** stream pointer, array of objects, count
- **Outputs/Return:** advanced stream pointer
- **Side effects:** reads/writes all monster_data fields in fixed order; asserts stream advance == Count * SIZEOF_monster_data
- **Calls:** StreamToValue()/ValueToStream() (macros for endian-aware I/O)
- **Notes:** Skips 7 shorts of padding at end. Used for save/load and multiplayer sync.

### XML_DamageKickParser::Start / HandleAttribute / AttributesDone / ResetValues
- **Purpose:** XML element parser for `<kick>` elements under `<damage_kicks>` root
- **Behavior:** Parses index and optional base/mult/vertical attributes; backs up originals on first parse; applies changes to damage_kick_definitions[]; ResetValues restores originals
- **Notes:** Only index is required. Allows runtime tuning of damage-kick physics.

## Control Flow Notes
- **Initialization**: initialize_monsters() resets global counters; initialize_monsters_for_new_level() marks all active monsters to recalculate paths
- **Per-tick frame**: move_monsters() called once per game tick, orchestrating all monster updates:
  1. Inactive monsters scan for targets (one per frame)
  2. Active monsters: apply polygon damage, animate, update physics, search targets/paths (rate-limited)
  3. Execute actions (move, attack, die) based on action state
  4. Decay civilian-kill counter
- **Activation**: activate_monster() (not shown but called elsewhere) initializes monster fields and sets ACTIVE flag
- **On death**: monster_died() ΓåÆ orphan projectiles, retarget others, then kill_monster() ΓåÆ removes object, marks slot free
- **Rendering**: Monster's object_data drives animation and rendering via animate_object()

## External Dependencies
- **Notable includes**: map.h, render.h, interface.h, effects.h, projectiles.h, player.h, platforms.h, scenery.h, SoundManager.h, items.h, media.h, flood_map.h, Packing.h, lua_script.h, monster_definitions.h
- **External globals**: monsters, monster_definitions, dynamic_world, static_world, objects, map_polygons, map_lines, map_endpoints, map_sides
- **External functions**: get_object_data(), get_polygon_data(), find_adjacent_polygon(), new_map_object(), remove_map_object(), animate_object(), play_object_sound(), new_effect(), delete_path(), flood_map(), find_closest_appropriate_target(), orphan_projectiles(), activate_nearby_monsters(), teleport_object_out(), L_Invalidate_Monster(), IsMediaDangerous()
- **Macros**: SLOT_IS_USED, MARK_SLOT_AS_USED, MONSTER_IS_ACTIVE, MONSTER_IS_DYING, etc. (from map.h and monsters.h)
