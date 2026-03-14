# Source_Files/GameWorld/weapons.h

## File Purpose
Header file for the weapons subsystem in the Aleph One game engine. Defines weapon types, data structures for tracking weapon state and ammunition, and functions for initializing, updating, and managing player weapons throughout gameplay.

## Core Responsibilities
- Enumerate weapon types (_weapon_fist, _weapon_pistol, _weapon_plasma_pistol, etc.)
- Define data structures for weapon state, triggers, shell casings, and display information
- Declare initialization functions for the weapon manager and per-player weapons
- Implement weapon update logic called once per frame with player actions
- Handle item pickups for ammunition/reloading and weapon switching
- Manage weapon display positioning and animation for the 3D weapon model
- Provide serialization (packing/unpacking) for multiplayer/save support
- Support XML-based weapon definition configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| weapon_display_information | struct | Position, animation frame, and rendering mode for 3D weapon model display |
| trigger_data | struct | State for one trigger (primary/secondary): ammo, firing phase, animation sequence |
| weapon_data | struct | Complete state for one weapon including both triggers and flags |
| shell_casing_data | struct | Physics data for ejected shell casings (position, velocity, type) |
| player_weapon_data | struct | Root container: current weapon, desired weapon, all weapons and shell casings for a player |
| Weapon enum | enum | Types: fist, pistol, plasma pistol, assault rifle, missile launcher, flamethrower, shotgun, SMG, etc. |
| Weapon action enum | enum | Animation states: idle, charging, firing |
| Trigger enum | enum | Primary and secondary trigger slots |
| Display positioning enum | enum | Positioning modes: low, center, high (for HUD placement) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| SIZEOF_weapon_definition | const int | global | Size of external weapon definition structure (134 bytes) for binary I/O |
| SIZEOF_player_weapon_data | const int | global | Size of player weapon data structure (472 bytes) for serialization |
| MAXIMUM_SHELL_CASINGS | enum | global | Max shell casings rendered simultaneously (4) |
| MAXIMUM_NUMBER_OF_WEAPONS | enum | global | Total weapon types the player can carry |

## Key Functions / Methods

### initialize_weapon_manager
- **Signature:** `void initialize_weapon_manager(void)`
- **Purpose:** One-time startup initialization of the weapon subsystem
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes global weapon state and resources
- **Calls:** Not inferable from this file
- **Notes:** Called once at application startup

### initialize_player_weapons_for_new_game
- **Signature:** `void initialize_player_weapons_for_new_game(short player_index)`
- **Purpose:** Reset all weapons to default state when starting a new game
- **Inputs:** `player_index` ΓÇö which player (index into player array)
- **Outputs/Return:** None
- **Side effects:** Zeros ammo, clears current weapon state
- **Calls:** Not inferable from this file
- **Notes:** Called during new game creation, before level load

### update_player_weapons
- **Signature:** `void update_player_weapons(short player_index, uint32 action_flags)`
- **Purpose:** Main per-frame weapon state update (firing, recoil, animation, ammo consumption)
- **Inputs:** `player_index`, `action_flags` (bitfield of player input actions)
- **Outputs/Return:** None
- **Side effects:** Modifies trigger state, ammo, animation frames, may create projectiles
- **Calls:** Not inferable from this file
- **Notes:** Core weapon loop; called once per frame for active player

### get_weapon_display_information
- **Signature:** `bool get_weapon_display_information(short *count, struct weapon_display_information *data)`
- **Purpose:** Query display parameters for rendering the 3D weapon model
- **Inputs:** `count` ΓÇö iteration counter; `data` ΓÇö output structure pointer
- **Outputs/Return:** `true` while iteration continues; caller keeps polling until `false`
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Iterator pattern; caller must call repeatedly to accumulate all display frames

### player_hit_target
- **Signature:** `void player_hit_target(short player_index, short weapon_identifier)`
- **Purpose:** Notify weapon system that a projectile hit (for ammo feedback, hits counter)
- **Inputs:** `player_index`, `weapon_identifier` (opaque ID returned when projectile was created)
- **Outputs/Return:** None
- **Side effects:** Updates hit counter and recoil state
- **Calls:** Not inferable from this file
- **Notes:** Called by physics/collision system when projectile impacts

### discharge_charged_weapons
- **Signature:** `void discharge_charged_weapons(short player_index)`
- **Purpose:** Fire all charged weapons when player dies
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Triggers weapons, modifies ammo
- **Calls:** Not inferable from this file
- **Notes:** Prevents energy weapons from disappearing with the player

### Packing/Unpacking Functions
- **Signatures:**
  - `uint8 *unpack_player_weapon_data(uint8 *Stream, size_t Count)`
  - `uint8 *pack_player_weapon_data(uint8 *Stream, size_t Count)`
  - `uint8 *unpack_weapon_definition(uint8 *Stream, size_t Count)`
  - `uint8 *pack_weapon_definition(uint8 *Stream, size_t Count)`
- **Purpose:** Binary serialization/deserialization for multiplayer sync and save files
- **Inputs/Outputs:** Byte stream pointer and count; returns updated stream pointer
- **Side effects:** Reads from or writes to stream; modifies in-memory weapon state
- **Notes:** Each returns pointer for chaining multiple unpack/pack calls

### Weapons_GetParser
- **Signature:** `XML_ElementParser *Weapons_GetParser()`
- **Purpose:** Return XML parser for weapon definition configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to XML element parser (defined elsewhere)
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Allows external XML loading of weapon stats and behavior

## Control Flow Notes
Weapon system is initialized at engine startup (`initialize_weapon_manager`), then per-player during player creation (`initialize_player_weapons`). The main update loop calls `update_player_weapons` once per frame with action flags from input/AI. Item pickups trigger `process_new_item_for_reloading`. Projectile impacts call `player_hit_target`. Rendering queries display parameters via `get_weapon_display_information`. On player death, `discharge_charged_weapons` fires remaining charged weapons. Shutdown and save/load use packing/unpacking functions.

## External Dependencies
- **Custom types:** `_fixed` (fixed-point integer), `uint16`, `uint32`, `uint8`, `size_t`
- **XML infrastructure:** `XML_ElementParser` (defined elsewhere, used in configuration)
- **Shell casing system:** `MAXIMUM_SHELL_CASINGS` constant exported to `lua_script.cpp` per inline comment
- **Weapon definition structure:** External binary size exposed as `SIZEOF_weapon_definition` for serialization without including the full definition
- **Resource management:** References weapon collections (collections are marked for load/unload but manager is not in this file)
