# Source_Files/GameWorld/monsters.h

## File Purpose
Header for monster/NPC management in a Marathon-engine game. Defines data structures for tracking monster state, enums for types/actions/modes, and function prototypes for lifecycle, activation, targeting, collision detection, and damage systems.

## Core Responsibilities
- Define 46+ monster types (ticks, fighters, hummers, cyborgs, enforcers, etc.)
- Declare monster state struct (`monster_data`, 64 bytes) tracking vitality, position, action, mode, target
- Define bit-packed flags for active/idle/berserk/blind/deaf/recovering-from-hit status
- Declare monster lifecycle functions (spawn, despawn, activate, deactivate)
- Declare AI/targeting functions (find target, change target, lock/lose lock)
- Declare damage and collision detection (radius damage, melee, projectile impacts)
- Declare activation triggers based on proximity, sound, teleport, or editor bias
- Provide macros for querying and mutating monster state bits

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `monster_data` | struct | 64-byte monster state: type, vitality, flags, path, action, mode, target, velocity, position, sound location, timers |
| Monster types enum | enum | 46+ types: marine (player), ticks, compilers, fighters, civilians, hummers, cyborgs, enforcers, hunters, troopers, juggernauts, defenders, yetis, tiny variants, fusion variants |
| Monster actions enum | enum | _stationary, _waiting_to_attack_again, _moving, _attacking_close/far, _being_hit, _dying_hard/soft/flaming, _teleporting (3 variants) |
| Monster modes enum | enum | _locked, _losing_lock, _lost_lock, _unlocked, _running (targeting/pathing state) |
| Monster flags enum | enum | _promoted, _demoted, _has_never_been_activated, _is_blind, _is_deaf, _teleports_out_when_deactivated |
| Activation bias enum | enum | _activate_on_player, _on_nearest_hostile, _on_goal, _randomly (editor-set behavior) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MonsterList` | `vector<monster_data>` | global | Dynamic array of all active monsters (replaces static array) |
| `monsters` | macro `(&MonsterList[0])` | global | Convenience pointer to first monster for legacy code |
| `MAXIMUM_MONSTERS_PER_MAP` | macro | global | Runtime limit from resource fork via `get_dynamic_limit()` |
| `LOCAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro | global | Dynamic limit for local collision checks |
| `GLOBAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro | global | Dynamic limit for global collision checks |

## Key Functions / Methods

### initialize_monsters / initialize_monsters_for_new_level
- **Purpose:** Lifecycle init; second called when map loads
- **Calls:** (defined elsewhere in monsters.c)

### move_monsters
- **Purpose:** Main update loop; assumes 1 tick per call
- **Called every frame:** Yes (frame/update cycle)

### new_monster / remove_monster
- **Purpose:** Spawn monster at location with type; despawn by index
- **Inputs:** `object_location`, monster type code
- **Outputs:** Monster index (or error code)

### activate_monster / deactivate_monster
- **Purpose:** Toggle active state; affects AI and rendering
- **Side effects:** Sets `MONSTER_IS_ACTIVE` flag; may trigger pathfinding

### find_closest_appropriate_target
- **Purpose:** Locate valid target for monster aggression
- **Inputs:** Aggressor index, full_circle search flag

### change_monster_target / get_monster_data
- **Purpose:** Reassign target; get mutable reference to monster state
- **Exposed to Pfhortran:** Yes (plugin support)

### damage_monster / damage_monsters_in_radius
- **Purpose:** Apply damage to single or multiple monsters
- **Inputs:** Target index, aggressor, damage definition, projectile index
- **Side effects:** Modifies vitality, triggers being-hit animation, may kill monster

### activate_nearby_monsters
- **Purpose:** Trigger monsters based on player proximity, sound, teleport, or trigger volume
- **Inputs:** Target index, caller index, activation flags (zone border, invisible, deaf, etc.)
- **Flags control:** Whether sound/trigger passes zone borders, uses editor biases, is unavoidable

### possible_intersecting_monsters
- **Purpose:** Find monsters (and scenery) in polygon for collision checks
- **Inputs:** Polygon index, optional count limit, scenery flag
- **Outputs:** Vector of object indices
- **Notes:** Growable list; `monsters_nearby()` convenience macro (no scenery)

### legal_monster_move / legal_player_move
- **Purpose:** Validate if location is walkable for NPC/player
- **Outputs:** Success flag; also returns floor height for player

### accelerate_monster
- **Purpose:** Apply velocity impulse (direction, elevation, magnitude)

### monster_died
- **Purpose:** Handle monster death (cleanup, target reassignment)

---

### Notes (Trivial Helpers)
- `get_monster_dimensions()` ΓÇö radius + height for collision
- `get_monster_impact_effect()` / `get_monster_melee_impact_effect()` ΓÇö death/hit animation codes
- `live_aliens_on_map()` ΓÇö win condition check
- `monster_placement_index()` / `placement_index_to_monster_type()` ΓÇö editor placement ID Γåö type mapping
- `try_to_add_random_monster()` ΓÇö spawn random monster at preset location
- `bump_monster()` ΓÇö collision bump response
- `legal_polygon_height_change()` / `adjust_monster_for_polygon_height_change()` ΓÇö floor/ceiling damage
- `monster_moved()` ΓÇö notify when monster crosses polygon boundary
- `mark_monster_collections()` / `load_monster_sounds()` ΓÇö asset management
- `SetPlayerViewAttribs()` ΓÇö for guided projectile target sight
- `unpack_monster_data()` / `pack_monster_data()` / `unpack_monster_definition()` / `pack_monster_definition()` ΓÇö serialization
- `init_monster_definitions()` ΓÇö data initialization
- `DamageKicks_GetParser()` ΓÇö XML parser factory for damage effects

## Control Flow Notes
- **Init phase:** `initialize_monsters()` ΓåÆ `initialize_monsters_for_new_level()` on map load
- **Update phase:** `move_monsters()` called once per game tick
- **Activation:** On-demand via trigger volumes, proximity, or sound propagation; controlled by flags and editor biases
- **Targeting:** Monsters lock onto targets and lose lock based on polygon changes; berserk monsters stick to geometrically closest target
- **Damage flow:** Projectiles/melee trigger `damage_monster()` ΓåÆ vitality check ΓåÆ being-hit animation (with recovery cooldown) ΓåÆ death if vitality Γëñ 0
- **Deactivation:** Monsters not in nearby polygons can be deactivated; may teleport out if flag set

## External Dependencies
- **dynamic_limits.h** ΓÇö `get_dynamic_limit()` for runtime limits on monster count, collision buffers
- **XML_ElementParser.h** ΓÇö `XML_ElementParser` base class for parsing monster definitions
- **cstypes.h** (implied) ΓÇö `int16`, `uint16`, `int32`, `world_distance`, `world_point3d`, etc.
- **\<vector\>** ΓÇö STL container for `MonsterList`
- **Constants:** `BUILD_DESCRIPTOR()` macro (collection/shape indices)
