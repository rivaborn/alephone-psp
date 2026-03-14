# Source_Files/GameWorld/projectiles.h

## File Purpose
Header for the projectile subsystem in the Aleph One game engine (Marathon-compatible). Defines data structures for active projectiles, enumerations of projectile types, and function prototypes for projectile creation, movement, collision detection, and cleanup.

## Core Responsibilities
- Enumerate all projectile types (rockets, grenades, bullets, energy weapons, etc.)
- Define the `projectile_data` structure for tracking active projectiles in the world
- Manage projectile lifecycle (creation, movement per frame, removal, detonation)
- Provide collision detection and obstruction reporting during projectile movement
- Handle projectile state flags (flyby events, damage causation)
- Serialize/deserialize projectile data and definitions for save/load
- Track owned vs. orphaned projectiles (whose owner died)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct | Active projectile instance (32 bytes): type, owner, target, position, velocity, damage, contrails. |
| (projectile type enum) | enum | Enumeration of ~40 projectile types (e.g. `_projectile_rocket`, `_projectile_smg_bullet`). |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ProjectileList` | `vector<projectile_data>` | global | Dynamic array of all active projectiles in the current map. |

## Key Functions / Methods

### new_projectile
- **Signature:** `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- **Purpose:** Create and add a new projectile to the world.
- **Inputs:** Origin point, starting polygon, velocity vector, rotation angle, projectile type, owner entity index/type, guided target, damage scaling.
- **Outputs/Return:** Index of the newly created projectile in `ProjectileList`.
- **Side effects:** Adds entry to `ProjectileList`; may allocate collection resources.
- **Calls:** (not visible in header; defined elsewhere).

### translate_projectile
- **Signature:** `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight)`
- **Purpose:** Move a projectile through the world one step; detect collisions and report obstruction type.
- **Inputs:** Projectile type, current and target 3D positions, polygon indices, owner, preflight flag.
- **Outputs/Return:** Bitmask of collision flags (`_projectile_hit*`); fills obstruction and line indices.
- **Side effects:** May modify polygon index if projectile crosses boundaries.
- **Calls:** (not visible; handles geometry queries and collision detection).
- **Notes:** Handles media boundary penetration; location of hit may differ from final position.

### move_projectiles
- **Signature:** `void move_projectiles(void)`
- **Purpose:** Update all active projectiles for one frame tick (╬öt = 1 tick).
- **Inputs:** None (operates on global `ProjectileList`).
- **Outputs/Return:** None.
- **Side effects:** Modifies positions/velocities of all projectiles; may trigger collisions and detonations; may remove projectiles from list.
- **Calls:** (not visible; calls `translate_projectile`, collision/damage handlers).
- **Notes:** Main per-frame update entry point for projectile subsystem.

### detonate_projectile
- **Signature:** `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)`
- **Purpose:** Trigger explosion/detonation of a projectile at a location (not necessarily its current position).
- **Inputs:** Detonation location, type, owner, damage.
- **Outputs/Return:** None.
- **Side effects:** Spawns damage effects/explosions; may damage nearby entities.

### remove_projectile / remove_all_projectiles
- **Purpose:** Remove single or all projectiles from `ProjectileList`.
- **Side effects:** Frees slots in the projectile array.

### preflight_projectile
- **Purpose:** Validate that a projectile can be created at the given location with the given trajectory before actually creating it.

### ProjectileIsGuided
- **Signature:** `bool ProjectileIsGuided(short Type)`
- **Purpose:** Query whether a projectile type is guided (seeks a target).

## Control Flow Notes
- **Initialization:** `init_projectile_definitions()` populates projectile type definitions from resources.
- **Per-frame:** `move_projectiles()` is called each game tick to advance all active projectiles, handle collisions, and trigger detonations.
- **Lifecycle:** Projectiles are created via `new_projectile()`, moved by the frame loop, collide via `translate_projectile()` (which reports obstruction types), and are removed by `remove_projectile()` or cleaned up in `remove_all_projectiles()` when the map unloads.
- **Owner tracking:** Projectiles remember their owner; `orphan_projectiles()` updates owned projectiles when an owner entity dies.

## External Dependencies
- **Includes:** `dynamic_limits.h` (for `get_dynamic_limit()`), `world.h` (for angle, world_point3d, world_distance types).
- **Defined elsewhere:** All function implementations in `PROJECTILES.C`; projectile definitions and sound resources (resource-fork or XML).
- **Types used:** `world_point3d`, `world_distance`, `angle`, `_fixed`, polygon indices, entity indices.
