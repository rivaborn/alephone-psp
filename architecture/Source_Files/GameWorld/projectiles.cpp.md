# Source_Files/GameWorld/projectiles.cpp

## File Purpose
Manages the lifecycle and physics of all active projectiles in the game world. Handles projectile creation with initial conditions, per-tick movement and collision detection against world geometry and entities, and detonation with area-of-effect damage and visual effects.

## Core Responsibilities
- **Projectile creation** ΓÇö `new_projectile()` allocates and initializes projectiles with type, owner, velocity, and guided-target information
- **Per-tick simulation** ΓÇö `move_projectiles()` updates all active projectiles, applies gravity, detects collisions, and removes projectiles on detonation
- **Collision detection** ΓÇö `translate_projectile()` traces projectile path through map geometry and checks against monsters/scenery via spatial queries
- **Guided targeting** ΓÇö `update_guided_projectile()` computes steering corrections toward a target monster each tick
- **Detonation** ΓÇö `detonate_projectile()` applies area-of-effect damage and spawns visual effects
- **Lifecycle management** ΓÇö `remove_projectile()`, `orphan_projectiles()` (when owners die), `remove_all_projectiles()` (on level exit)
- **Media boundary handling** ΓÇö special case for projectiles with "penetrates media boundary" flag allowing water-surface interactions
- **Serialization** ΓÇö pack/unpack functions for save/restore and network synchronization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct | Instance state: type, owner, target, elevation, gravity, position, contrail counters, damage scale |
| `projectile_definition` | struct | Static definition: speed, radius, effects, flags (guided, rebounds, persistent, etc.) |
| `IntersectedObjects` (static) | `vector<short>` | Workspace for collision checks; reused across all projectiles in one frame |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `IntersectedObjects` | `vector<short>` | static | Growable list of object indices (monsters/scenery) in current polygon for collision tests |
| `alien_projectile_override` | `short` | global | Projectile type substitution if copy-protection fails |
| `human_projectile_override` | `short` | global | Projectile type substitution if copy-protection fails |

## Key Functions / Methods

### new_projectile
- Signature: `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- Purpose: Allocate and initialize a projectile in the world
- Inputs: Origin location/polygon, velocity vector, type, owner identity, optional guided target, damage scaling
- Outputs/Return: Index into projectiles array (or NONE if no free slots)
- Side effects: Allocates a map object; marks projectile slot as used; potentially adjusts type based on media
- Calls: `adjust_projectile_type()`, `get_projectile_definition()`, `new_map_object3d()`, `GET_OBJECT_DATA()`, `normalize_angle()`
- Notes: Adjusts firing angle by delta_theta if projectile definition permits; handles projectile-type substitution via alien/human overrides; contrail/flyby state initialized to zero

### move_projectiles
- Signature: `void move_projectiles(void)`
- Purpose: Main per-tick update for all active projectiles
- Inputs: None (operates on global projectiles array)
- Outputs/Return: None (modifies projectile state in-place)
- Side effects: Moves projectile objects, may play sounds, spawn effects, apply damage, remove projectiles
- Calls: `animate_object()`, `update_guided_projectile()`, `translate_projectile()`, `remove_projectile()`, `damage_scenery()`, `damage_monster()`, `damage_monsters_in_radius()`, `new_effect()`, `play_object_sound()`, Lua callbacks
- Notes: Handles gravity, wander, guided targeting; checks animation-stop conditions; applies rebound logic; generates contrails; tracks distance for range checks

### translate_projectile
- Signature: `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight)`
- Purpose: Trace projectile path from old to new location, detect collisions with map geometry and entities
- Inputs: Projectile type, start/end positions and polygon, owner index, preflight flag
- Outputs/Return: Flags indicating what was hit; out-params for new polygon and obstruction object index; return value combines `_projectile_hit*` bits
- Side effects: May call `try_and_toggle_control_panel()` if projectile hits a switch
- Calls: `get_polygon_data()`, `get_media_data()`, `find_line_crossed_leaving_polygon()`, `find_line_intersection()`, `find_adjacent_polygon()`, `possible_intersecting_monsters()`, `GET_OBJECT_OWNER()`, `get_monster_dimensions()`, `get_scenery_dimensions()`, `distance2d()`, `point_to_line_segment_distance_squared()`
- Notes: Loop exits when hitting something or reaching destination; special handling for "penetrates media boundary" flag via `traveled_underneath` logic; accounts for adjacent-polygon geometry when testing line crossings

### update_guided_projectile
- Signature: `static void update_guided_projectile(short projectile_index)`
- Purpose: Compute steering corrections for guided projectiles toward their target
- Inputs: Projectile index
- Outputs/Return: None (modifies projectile facing/elevation)
- Side effects: Updates projectile object facing and projectile elevation
- Calls: `get_projectile_data()`, `get_monster_data()`, `get_object_data()`, `get_monster_dimensions()`
- Notes: Difficulty level affects steering speed; invisible targets can be locked only on _total_carnage_level; uses trigonometric tables (sine/cosine) for angle arithmetic

### preflight_projectile
- Signature: `bool preflight_projectile(world_point3d *origin, short origin_polygon_index, world_point3d *destination, angle delta_theta, short type, short owner, short owner_type, short *obstruction_index)`
- Purpose: Validate a proposed projectile before firing; detect pre-spawn obstructions
- Inputs: Origin, destination, type, owner, firing error angle
- Outputs/Return: true if legal (unobstructed), false otherwise; out-param for hit object
- Side effects: None
- Calls: `get_projectile_definition()`, `get_polygon_data()`, `get_media_data()`, `translate_projectile()`, `arctangent()`, `isqrt()`
- Notes: Rejects if firing angle exceeds MAXIMUM_PROJECTILE_ELEVATION; handles media penetration checks

### detonate_projectile
- Signature: `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)`
- Purpose: Trigger detonation effects (area-of-effect damage and visual effects)
- Inputs: Detonation location, type, owner, damage scaling
- Outputs/Return: None
- Side effects: Damages monsters in radius; spawns visual effect; calls Lua callback
- Calls: `get_projectile_definition()`, `damage_monsters_in_radius()`, `new_effect()`, `L_Call_Projectile_Detonated()`
- Notes: Only meaningful for area-of-effect projectiles

## Control Flow Notes
Called from `move_projectiles()` (game loop per-tick update). Projectiles are created on-demand by weapon/monster code and removed when detonating, reaching max range, or when their owner dies. The per-tick cycle is: animate ΓåÆ apply gravity/guidance ΓåÆ move via `translate_projectile()` ΓåÆ check for detonation or removal ΓåÆ leave contrail.

## External Dependencies
- **Geometry**: `map.h` (polygons, lines, endpoints, adjacent-polygon queries)
- **Entities**: `monsters.h`, `scenery.h`, `effects.h` (damage, visual effects)
- **Objects**: Map object creation/manipulation (`new_map_object3d`, `translate_map_object`, `remove_map_object`)
- **Projectile definitions**: `projectile_definitions.h` (imported; not defined here)
- **Sound**: `SoundManager.h` (flyby/rebound sounds)
- **Scripting**: `lua_script.h` (Lua detonation callbacks)
- **Utilities**: `cseries.h`, `Packing.h`, `dynamic_limits.h`
