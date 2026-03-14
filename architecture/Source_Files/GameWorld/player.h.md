# Source_Files/GameWorld/player.h

## File Purpose
Defines core player data structures, action flags, physics variables, and the API for managing player state in the Marathon/Aleph One engine. Covers player initialization, physics updates, weapons, damage, and persistent game state.

## Core Responsibilities
- Define `player_data` structure encapsulating all player state (location, energy, weapons, damage, inventory)
- Define `physics_variables` for per-player kinematics (position, velocity, heading, floor/ceiling heights)
- Manage player settings (initial energy, oxygen, visual range, team colors) via MML-configurable `player_settings_definition`
- Expose action flag bit definitions and macros for input encoding (movement, aiming, weapon triggers)
- Provide global player array and helper accessors (`local_player`, `current_player`)
- Declare player lifecycle functions (creation, deletion, per-tick updates)
- Declare physics and weapon update routines; pack/unpack for serialization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_data` | struct | Complete player state: identity, location, energy/oxygen, weapons, items, damage history, teleport phase, interface state |
| `physics_variables` | struct | Per-player kinematics: position, velocity, heading, elevation, floor/ceiling heights, step phase, media/ground contact flags |
| `player_settings_definition` | struct | Tunable gameplay parameters (initial energy/oxygen, powerup bonuses, visual range, oxygen depletion rates, invincibility vulnerability) |
| `player_shape_definitions` | struct | Collection of animation shape indices for player model (legs, torsos, firing/charging states, death animations) |
| `damage_record` | struct | Aggregate damage dealt/taken: total damage amount and kill count |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_settings` | `struct player_settings_definition` | extern | Runtime-configurable player parameters (damage, behavior, physics tuning) |
| `players` | `struct player_data *` | extern | Dynamic array of all active players in the game |
| `local_player_index`, `current_player_index` | `short` | extern | Indices into `players` array; use setters to change |
| `local_player`, `current_player` | `struct player_data *` | extern | Convenience pointers (shadows indices) |
| `team_damage_given[NUMBER_OF_TEAM_COLORS]` | `struct damage_record` | extern | Aggregate team-level damage stats |
| `team_damage_taken[NUMBER_OF_TEAM_COLORS]` | `struct damage_record` | extern | Aggregate team damage received |
| `GetRealActionQueues()` | function returning `ActionQueues*` | extern | Accessor for current input action queue (avoids accidental pointer assignment) |

## Key Functions / Methods

### initialize_players
- **Signature:** `void initialize_players(void)`
- **Purpose:** One-time engine initialization for the player subsystem
- **Side effects:** Allocates memory, sets up state arrays
- **Calls:** (inferred) `allocate_player_memory()`

### new_player
- **Signature:** `short new_player(short team, short color, short player_identifier)`
- **Purpose:** Create a new player and assign to the player array
- **Return:** Player index (or NONE on failure)
- **Side effects:** Modifies the global `players` array, increments player count

### delete_player
- **Signature:** `void delete_player(short player_number)`
- **Purpose:** Remove a player from the game
- **Inputs:** `player_number` ΓÇô index into `players` array
- **Side effects:** Clears player slot, adjusts player count

### update_players
- **Signature:** `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- **Purpose:** Advance all players' state by one tick (movement, weapon cooldown, item use, etc.)
- **Inputs:** `inActionQueuesToUse` ΓÇô action queue to consume input from; `inPredictive` ΓÇô if true, performs lookahead without modifying persistent state
- **Side effects:** Updates physics, energy, oxygen, weapon state, damage tracking; may spawn projectiles or effects
- **Notes:** Ticks-per-frame is 1; supports rollback-safe "predictive" updates for netplay

### damage_player
- **Signature:** `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- **Purpose:** Apply damage from a projectile or melee hit to a player
- **Inputs:** `monster_index` ΓÇô victim player's monster slot; `aggressor_index` ΓÇô attacker ID; `damage` ΓÇô damage type and amount; `projectile_index` ΓÇô optional projectile reference
- **Side effects:** Decrements suit energy; broadcasts damage record; may trigger death
- **Calls:** Weapon/projectile manager (implicit)

### get_player_data
- **Signature:** `player_data *get_player_data(const size_t player_index)`
- **Purpose:** Bounds-checked access to a player structure
- **Return:** Pointer to `players[player_index]`, or NULL if out of bounds
- **Notes:** Replaces unsafe direct array access

### update_player_physics_variables
- **Signature:** `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- **Purpose:** Integrate physics for one player each tick (movement, gravity, collisions, floor/ceiling tracking)
- **Inputs:** `action_flags` ΓÇô bit flags encoding input (turning, moving, jumping, etc.); `predictive` ΓÇô rollback-safe mode
- **Side effects:** Updates position, velocity, heading, elevation, supporting polygon; may trigger landing/liquid-entry effects

### initialize_player_physics_variables
- **Signature:** `void initialize_player_physics_variables(short player_index)`
- **Purpose:** Reset a player's physics state (e.g., on level load or respawn)
- **Side effects:** Clears velocity, resets heading, queries initial floor height

### instantiate_absolute_positioning_information
- **Signature:** `void instantiate_absolute_positioning_information(short player_index, _fixed facing, _fixed elevation)`
- **Purpose:** Apply absolute camera positioning (e.g., from network sync or recorded input)
- **Inputs:** `facing` ΓÇô yaw angle; `elevation` ΓÇô pitch angle
- **Side effects:** Overwrites player heading and elevation (used in netplay or scripting)

### mask_in_absolute_positioning_information
- **Signature:** `uint32 mask_in_absolute_positioning_information(uint32 action_flags, _fixed yaw, _fixed pitch, _fixed velocity)`
- **Purpose:** Encode absolute positioning data into action flags bit-field
- **Return:** Modified action flags with positioned data embedded
- **Notes:** Inverse of `instantiate_absolute_positioning_information`; enables network compression

### find_action_key_target
- **Signature:** `short find_action_key_target(short player_index, world_distance range, short *target_type)`
- **Purpose:** Detect nearby interaction targets (platforms, switches, teleporters) within range of the player
- **Return:** Index of target (e.g., polygon or control panel), or NONE
- **Outputs:** `target_type` set to `_target_is_platform`, `_target_is_control_panel`, or `_target_is_unrecognized`

### set_local_player_index / set_current_player_index
- **Signature:** `void set_local_player_index(short player_index); void set_current_player_index(short player_index)`
- **Purpose:** Change which player is "local" (controlled by this client) or "current" (being updated)
- **Side effects:** Updates both index and pointer shadows
- **Notes:** Must use setters; direct assignment to indices is unsafe

### player_identifier_to_player_index
- **Signature:** `short player_identifier_to_player_index(short player_identifier)`
- **Purpose:** Map a persistent player ID to current array index
- **Return:** Index into `players` array, or NONE if not found

### unpack_player_data / pack_player_data
- **Signature:** `uint8 *unpack_player_data(uint8 *Stream, player_data *Objects, size_t Count); uint8 *pack_player_data(uint8 *Stream, player_data *Objects, size_t Count)`
- **Purpose:** Serialize player state to/from byte streams (for saves or network)
- **Return:** Updated stream pointer
- **Notes:** Defined elsewhere; allows external code to preserve player state

---

## Control Flow Notes

**Initialization phase:**
`initialize_players()` ΓåÆ `allocate_player_memory()` ΓåÆ optionally `initialize_player_physics_variables()` for each player.

**Per-tick update (main loop):**
`update_players()` is called with action input each frame; internally calls `update_player_physics_variables()` and weapon/damage routines.

**Level load:**
`recreate_players_for_new_level()` resets physics and weapons; `initialize_map_for_new_player()` places players at spawn points.

**Damage/death:**
`damage_player()` is triggered by collision or projectile hit; may set `_player_is_dead_flag` or `_player_is_totally_dead_flag`.

**Serialization:**
Pack/unpack functions integrate with save-game and network synchronization pipelines.

## External Dependencies

- **cseries.h** ΓÇô Base types, macros, memory utilities
- **world.h** ΓÇô Coordinate types (`world_point3d`, `angle`, `world_distance`, `_fixed`), trigonometry
- **map.h** ΓÇô World geometry accessors, damage definitions, game mode constants
- **XML_ElementParser.h** ΓÇô MML parser for runtime configuration
- **weapons.h** ΓÇô Weapon enums (`_weapon_pistol`, etc.) and weapon update routines
- **Implicit:** Monster/projectile managers (called during damage and update), platform/switch managers (called during action-key handling)
