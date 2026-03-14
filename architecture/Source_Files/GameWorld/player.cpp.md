# Source_Files/GameWorld/player.cpp

## File Purpose
Core player management system for the Aleph One game engine. Handles player lifecycle (creation, update, death/revival), state management (physics, inventory, powerups), damage/health, teleportation, serialization, and XML configuration of player mechanics. Approximately 2883 lines implementing the most frequently-updated game object type.

## Core Responsibilities
- Player creation, initialization, and teardown
- Per-tick player updates: physics, actions, powerups, oxygen mechanics
- Damage application, death sequences, respawning with penalty delays
- Weapon state management (intensity, decay)
- Oxygen depletion/replenishment based on environment (vacuum vs. normal air)
- Player shape/animation selection based on action state
- Teleportation logic (intra-level and inter-level)
- Powerup duration tracking and application (invisibility, invincibility, infravision, extravision)
- Item pickup and inventory management
- Player data serialization for save-games and network sync
- XML-driven configuration of player parameters and item definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| player_data | struct | Complete player runtime state: health, location, items, weapons, powerups, flags |
| physics_variables | struct | Player movement/animation state: position, velocity, facing, elevation, step phase |
| player_settings_definition | struct | Configurable gameplay values: energy/oxygen levels, self-luminosity, oxygen rates, swimming, guided missiles |
| player_powerup_durations_definition | struct | Duration (in ticks) for each powerup type |
| player_shape_definitions | struct | Shape indices for player body animation: legs, idle/charging/firing torsos |
| damage_response_definition | struct | Damage type ΓåÆ (fade effect, sound, death animation) mapping |
| damage_record | struct | Damage statistics: total damage, kill count |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| players | player_data* | global | Dynamic array of all player instances |
| local_player, current_player | player_data* | global | Fast pointers to local/spectated player |
| team_damage_given, team_damage_taken, team_monster_damage_* | damage_record[8] | global | Per-team PvP/PvE damage tracking for stats |
| sRealActionQueues | ActionQueues* | static | Input queue system; shared with network/recording |
| player_shapes | player_shape_definitions | static | Collection 6 shape indices for player body |
| player_initial_items[] | short[] | static | Default starting inventory (magnum, ammo, knives) |
| damage_response_definitions[] | damage_response_definition[] | static | Lookup table: damage type ΓåÆ visual/audio response |
| player_powerup_durations | player_powerup_durations_definition | static | Tick durations (configurable via XML) |
| player_settings | player_settings_definition | static | Gameplay parameters (energy, oxygen, pools) |
| player_powerups | player_powerup_definition | static | Item ID ΓåÆ powerup type mapping |
| sLocalPlayerTicksSinceTerminal | int | static | Tick counter for terminal-mode state tracking |

## Key Functions / Methods

### allocate_player_memory
- Signature: `void allocate_player_memory(void)`
- Purpose: Pre-game initialization; allocate player array and action queues
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates global `players` array and `sRealActionQueues`
- Calls: C++ `new`
- Notes: Called once at engine startup; size determined by `MAXIMUM_NUMBER_OF_PLAYERS`

### new_player
- Signature: `short new_player(short team, short color, short identifier)`
- Purpose: Create and initialize a new player
- Inputs: Team (0ΓÇô7), color index, identifier flags (auto-recenter, auto-switch-weapons)
- Outputs/Return: Player index (0-based)
- Side effects: Increments `dynamic_world->player_count`; initializes physics, powerups, inventory
- Calls: `get_player_data()`, `recreate_player()`, `initialize_player_weapons()`, `give_player_initial_items()`
- Notes: Sets teleport destination to `NO_TELEPORTATION_DESTINATION` (INT16_MAX); clears inventory

### update_players
- Signature: `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- Purpose: Per-tick main update loop; process all players' actions, physics, state changes
- Inputs: ActionQueues (input source), predictive flag (network prediction mode)
- Outputs/Return: None (modifies player state in-place)
- Side effects: Updates position, facing, health, oxygen, powerups; may kill/revive; plays sounds; marks dirty items
- Calls: `dequeueActionFlags()`, `update_player_physics_variables()`, `swipe_nearby_items()`, `update_player_weapons()`, `update_player_teleport()`, `update_player_media()`, `set_player_shapes()`, `ReplenishPlayerOxygen()`, `handle_player_in_vacuum()`
- Notes: Core 30-Hz tick; handles ball mechanics, media detection, microphone, terminal mode, oxygen, extravision decay

### damage_player
- Signature: `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- Purpose: Apply damage from an external source
- Inputs: Target, aggressor, damage definition
- Outputs/Return: None
- Side effects: Decrements health; triggers fade effect; may kill player
- Calls: `kill_player()`, `play_object_sound()`
- Notes: Respects invincibility powerups; clamps health to [0, max]

### kill_player / revive_player
- Signature: `static void kill_player(short, short, short)` / `static void revive_player(short player_index)`
- Purpose: Initiate death sequence / respawn player
- Side effects: Sets dead flags / clears flags, resets physics, resets view
- Calls: `set_player_dead_shape()`, `initialize_player_physics_variables()`, `ResetFieldOfView()`
- Notes: Respawn blocked by `reincarnation_delay` if penalties enabled

### get_player_data
- Signature: `player_data *get_player_data(const size_t player_index)`
- Purpose: Safe bounds-checked player data accessor
- Inputs: Player index
- Outputs/Return: Pointer to player struct or vassert failure
- Calls: `GetMemberWithBounds()`, `vassert()`

### pack_player_data / unpack_player_data
- Signature: `uint8 *pack/unpack_player_data(uint8 *Stream, player_data *Objects, size_t Count)`
- Purpose: Serialize/deserialize player state (save-games, network sync)
- Inputs: Byte stream, player array, count
- Outputs/Return: Advanced stream pointer
- Side effects: Stream modified (written to or read from)
- Calls: `ValueToStream()`, `ListToStream()`, `BytesToStream()` and reverse operations

### reset_action_queues / reset_player_queues
- Signature: `void reset_action_queues(void)` / `static void reset_player_queues(void)`
- Purpose: Clear queued input and recording state between levels
- Side effects: Flushes `sRealActionQueues` and recording/playback buffers
- Notes: Called when loading a map to prevent stale input garbage

### handle_player_in_vacuum
- Signature: `static void handle_player_in_vacuum(short player_index, uint32 action_flags)`
- Purpose: Apply oxygen depletion and vacuum/liquid breathing rules
- Inputs: Player index, action flags (for run/fire oxygen burn multiplier)
- Outputs/Return: None
- Side effects: Decrements oxygen; plays warning sound at low levels; kills player if exhausted
- Calls: `kill_player()`, `play_object_sound()`

## XML Parser Classes

Nested classes inheriting `XML_ElementParser` for configurable player mechanics:
- `XML_StartItemParser` ΓÇö initial inventory items
- `XML_PlayerDamageParser` ΓÇö damage type ΓåÆ response (fade/sound)
- `XML_PowerupParser` ΓÇö powerup durations
- `XML_PowerupAssignParser` ΓÇö powerup item IDs
- `XML_PlayerShapeParser` ΓÇö animation shapes
- `XML_PlayerParser` ΓÇö player settings

All follow standard pattern: `Start()`, `HandleAttribute()`, `AttributesDone()`, `ResetValues()`.

## Control Flow Notes
**Init**: `allocate_player_memory()` ΓåÆ `initialize_players()` ΓåÆ `new_player()` (per player).  
**Frame**: `update_players()` called at 30 Hz; dequeues actions, updates physics/powerups, applies oxygen/media, kills/revives.  
**Death**: `kill_player()` initiates; `revive_player()` respawns (delayed by penalty).  
**Save/Load**: `pack_player_data()` to disk; `unpack_player_data()` from disk.

## External Dependencies
- `map.h`, `world.h`: World geometry, polygons, coordinates
- `monsters.h`, `monster_definitions.h`: Monster class/damage types
- `weapons.h`, `items.h`: Weapon/item enums
- `SoundManager.h`, `fades.h`, `media.h`: Audio and visual effects
- `interface.h`, `game_window.h`: UI/view management
- `ActionQueues.h`, `Packing.h`: Input queues, serialization helpers
- `ChaseCam.h`, `lua_script.h`: Optional subsystems (chase camera, scripting)
- `XML_ElementParser.h`: Configuration parsing
