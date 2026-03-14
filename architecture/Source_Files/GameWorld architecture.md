# Subsystem Overview

## Purpose
GameWorld is the core simulation engine managing all dynamic game entities (players, monsters, items, projectiles, effects, platforms) and world geometry for the Marathon game engine on PSP. It orchestrates the main per-tick game loop, performs spatial queries and collision detection on the polygon-based terrain, and coordinates updates across physics, AI, rendering, and sound subsystems. This subsystem bridges SDL/libGU platform abstraction with high-level gameplay logic while maintaining deterministic state for networked play.

## Key Files

| File | Role |
|------|------|
| **marathon2.cpp** | Core game initialization, main update loop (`update_world()`), entity lifecycle management, action queue coordination |
| **map.cpp / map.h** | Central world/geometry management: allocation, object creation/destruction, polygon/line/endpoint storage, spatial queries, collision testing, sound propagation |
| **map_constructors.cpp** | Map geometry initialization: polygon/line/endpoint construction, derived property recalculation, serialization (pack/unpack) |
| **world.cpp / world.h** | Low-level math primitives: trigonometry lookup tables, coordinate transformations, distance calculations, angle arithmetic, random number generation |
| **player.cpp / player.h** | Player lifecycle, state management, inventory, damage/health, physics variables, serialization, XML configuration |
| **physics.cpp** | Player physics simulation: movement input, collision detection, gravity, jumping, camera positioning, external forces |
| **physics_models.h** | Physics parameter constants for locomotion modes (walking/running) and serialization utilities |
| **monsters.cpp / monsters.h** | Monster/NPC lifecycle, AI behavior, targeting, pathfinding, combat, animation sequencing, damage, serialization |
| **monster_definitions.h** | Static monster type configuration: behavioral flags, speed, attack definitions, item drops |
| **pathfinding.cpp** | Path management and traversal using flood-fill algorithm; path generation and movement waypoint calculation |
| **flood_map.cpp / flood_map.h** | Flood-fill/breadth-first pathfinding implementation; search node management and path reconstruction |
| **projectiles.cpp / projectiles.h** | Projectile lifecycle: creation with initial conditions, per-tick movement/collision detection, detonation with area damage, guided targeting |
| **projectile_definitions.h** | Static projectile type configuration: damage, visual effects, physics parameters, behavioral flags |
| **effects.cpp / effects.h** | Temporary visual/audio effects management: creation, animation updates, removal, serialization |
| **effect_definitions.h** | Static effect metadata: animation sequences, sound associations, detonation behavior |
| **items.cpp / items.h** | Item lifecycle: spawning, pickup mechanics, inventory tracking, animation, environment-specific availability, XML configuration |
| **item_definitions.h** | Static item metadata: types, shapes, spawn limits, environment restrictions |
| **platforms.cpp / platforms.h** | Platform (door/moving surface) simulation: creation, movement with collision/obstruction, player/monster interaction, state management, audio, serialization |
| **platform_definitions.h** | Static platform type configuration: sound mappings, behavior flags, damage properties |
| **lightsource.cpp / lightsource.h** | Dynamic light management: creation, animation, state transitions, intensity updates, serialization, legacy format support |
| **media.cpp / media.h** | Hazardous liquid/media management: creation, height/flow updates, damage/sound/effect queries, serialization, XML configuration |
| **media_definitions.h** | Static media type properties: visual representation, damage, detonation effects, audio |
| **scenery.cpp / scenery.h** | Static scenery objects: creation, animation, destruction handling, property queries, XML configuration |
| **scenery_definitions.h** | Static scenery metadata: collision radius, height, destruction effects, behavior flags |
| **devices.cpp** | Interactive control panels/switches: state management, player activation, sound/texture updates, light/platform cascading, network save coordination |
| **placement.cpp** | Dynamic entity placement/respawning: load configuration from map, maintain minimum object counts, validate safe spawn locations |
| **weapons.cpp / weapons.h** | Weapon state machines: firing/reloading/charging, trigger handling, ammunition tracking, shell casing effects, animation, serialization, XML configuration |
| **weapon_definitions.h** | Static weapon configuration: classes, behavior flags, animation states, trigger mechanics, shell physics |
| **dynamic_limits.cpp / dynamic_limits.h** | Configurable runtime entity limits (objects, monsters, projectiles, effects); array resizing, XML parsing |
| **editor.h** | Editor constants and version macros; map format compatibility definitions |
| **TickBasedCircularQueue.h** | Lock-free circular queue template for tick-keyed action buffers (networked prediction) |

## Core Responsibilities

- **Entity lifecycle management** ΓÇö allocate, initialize, update per-tick, and remove all dynamic objects (players, monsters, items, projectiles, effects, platforms, lights, media, scenery)
- **Game loop orchestration** ΓÇö coordinate per-frame/per-tick updates across physics, AI, animation, collision detection, damage processing, and serialization
- **Spatial queries and collision detection** ΓÇö polygon-based geometry representation, line-of-sight testing, point-in-polygon queries, line crossing detection, adjacent polygon traversal
- **World initialization and persistence** ΓÇö allocate map memory structures, load/unpack map data, manage level transitions, save/restore game state for save-games and network synchronization
- **Player state management** ΓÇö physics simulation, input processing, inventory, weapons, damage/death/respawn, oxygen mechanics, powerups, teleportation
- **AI and pathfinding** ΓÇö monster targeting, mode/action state machines, flood-fill-based path generation, waypoint navigation, attack execution
- **Environmental simulation** ΓÇö dynamic lighting, hazardous media (liquids), moving platforms/doors, visual effects, particle/shell casings, ambient sounds
- **Configuration and tuning** ΓÇö XML-driven dynamic entity limits, item properties, media behavior, weapon parameters, player settings, platform definitions
- **Network and prediction support** ΓÇö tick-based action queues, entity serialization, deterministic physics, game state snapshots for client-side prediction

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `update_world()` ΓÇö main per-tick entry point (called from game loop)
- `new_map_object()`, `remove_map_object()`, `get_object_data()` ΓÇö object creation/destruction/query
- `get_polygon_data()`, `get_player_data()`, `get_monster_data()` ΓÇö entity accessors
- `get_line_data()`, `get_endpoint_data()` ΓÇö map geometry accessors
- `find_adjacent_polygon()`, `find_line_crossed_leaving_polygon()` ΓÇö spatial queries
- `legal_player_move()`, `keep_line_segment_out_of_walls()` ΓÇö collision checking
- Player/monster/item/projectile update entry points called during main loop
- Global arrays: `map_polygons[]`, `ObjectList`, `MonsterList`, `ProjectileList`, `EffectList`, `LightList`, `MediaList` ΓÇö entity storage

**Consumes from other subsystems:**
- **Input** ΓÇö `ActionQueues.h` for player control input (movement, aiming, weapon triggers)
- **Rendering** ΓÇö `render.h` for camera/frustum information; entity shape/animation queries via `interface.h`
- **Sound** ΓÇö `SoundManager.h` for audio playback (weapon fire, impacts, ambient); obstruction callbacks for sound propagation
- **Platform/OS** ΓÇö `SDL.h` (via SDL 1.2 PSP port); `libGU` GPU commands for platforms.cpp (if GPU rendering enabled)
- **Scripting** ΓÇö `lua_script.h` for Lua hook callbacks (item creation, monster activation, light state changes)
- **Network** ΓÇö `network_games.h`, `ActionQueues.h` for multiplayer synchronization and client prediction
- **Configuration** ΓÇö `XML_ElementParser.h` for parsing MML/XML definitions of items, weapons, media, platforms, scenery

## Runtime Role

**Initialization:** 
- Game startup calls subsystem initialization functions (weapon manager, physics model setup, effect definitions loading)
- Level entry allocates map memory, unpacks map geometry from WAD file, initializes map objects, marks texture collections for loading, spawns initial monsters/items per placement rules

**Per-frame update:**
- `update_world()` called once per tick from main game loop with current action flags
- Processes player input ΓåÆ updates player physics/weapons/animation
- Updates all monsters (behavior, targeting, pathfinding, movement, animation)
- Moves projectiles with collision detection and detonation
- Updates effects (animation advance, lifetime countdown), lights (intensity cycling), media (height/flow)
- Updates platforms (movement with obstruction detection)
- Checks polygon triggers (damage zones, monster activation, objective conditions)
- Handles damage application, item pickup, device activation
- Calls Lua script hooks for entity lifecycle events
- Serializes predicted game state for networked clients
- Calls sound/rendering subsystems for audio/visual updates

**Shutdown:**
- Level exit clears all dynamic entities, unloads texture collections
- Game exit deallocates map memory structures

## Notable Implementation Details

- **Polygon-based spatial representation** ΓÇö game world divided into convex polygons with directed edges (sides), enabling efficient traversal and collision via adjacency and line crossing
- **Flood-fill pathfinding** ΓÇö AI navigation uses iterative breadth-first/best-first search over polygon adjacency with optional cost weighting; supports both destination-driven and random exploration
- **Fixed-point arithmetic** ΓÇö most coordinates and physics use `_fixed` (likely 16.16 format) for deterministic cross-platform behavior without floating-point variation
- **Tick-based synchronization** ΓÇö game loop driven by discrete ticks; action queues buffer input across frames for networked client prediction
- **Dynamic entity limits** ΓÇö object/monster/projectile/effect caps configurable via XML parser; entity arrays automatically resize when limits change
- **Serialization throughout** ΓÇö all major data structures support pack/unpack functions for save-games and network replication; legacy Marathon 1 map format supported via compatibility functions
- **Slot-based object storage** ΓÇö entities stored in vectors with slot occupancy flags (`SLOT_IS_USED`, `MARK_SLOT_AS_USED`); removed entities marked as free and recycled
- **Cascade effects** ΓÇö platform/light activation can trigger adjacent platforms or linked lights via static flags; device state changes propagate to connected features
- **Immediate-mode geometry updates** ΓÇö map geometry properties (polygon heights, line endpoints, side textures) updated in-place when platforms move; no deferred rebuild
- **Lua integration** ΓÇö scriptable hooks for entity creation/activation/damage events; XML MML system provides runtime reconfiguration of item/weapon/media/platform definitions without recompilation
