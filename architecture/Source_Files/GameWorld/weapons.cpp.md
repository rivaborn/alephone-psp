# Source_Files/GameWorld/weapons.cpp

## File Purpose
Implements the weapon system for the Marathon game engine, managing player weapon states, firing mechanics, reloading, ammunition tracking, animation sequences, and shell casing visual effects. Supports dynamic weapon configuration via XML and handles serialization of weapon data for save/network games.

## Core Responsibilities
- Initialize and reset weapon state for players (new games, level transitions)
- Update weapon state machines each frame (firing, reloading, charging, idle animations)
- Handle trigger input and coordinate firing/reloading logic
- Manage ammunition counts and two-handed/dual-trigger weapon mechanics
- Generate and update shell casing visual effects
- Track weapon statistics (shots fired, shots hit)
- Render weapon display information for HUD and weapon sprite positioning
- Parse and apply XML weapon configuration (definitions, shell casings, weapon ordering)
- Pack and unpack weapon data to/from network/save-game streams
- Handle weapon luminosity contribution to player light intensity

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `weapon_data` | struct | State of a specific weapon type for a player (flags, trigger states, etc.) |
| `trigger_data` | struct | Per-trigger state: animation sequence, rounds, firing ticks, shell casing tracking |
| `shell_casing_data` | struct | Visual shell casing particle: type, position, velocity, animation frame |
| `player_weapon_data` | struct | Complete weapon state for one player: current/desired weapon, all weapons, shell casings |
| `weapon_definition` | struct (extern) | Static weapon properties: firing intensity, shapes, reload timing, trigger definitions |
| `trigger_definition` | struct (extern) | Trigger properties: ammo type, timing, projectile type, sounds |
| `shell_casing_definition` | struct (extern) | Shell casing properties: initial position/velocity, acceleration |
| `weapon_display_information` | struct | HUD rendering data: shape, position, transfer mode, animation frame |
| `XML_ShellCasingParser` | class | XML parser for shell casing configuration |
| `XML_WeaponOrderParser` | class | XML parser for weapon priority ordering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_weapons_array` | `player_weapon_data*` | static | Array of weapon state for all players in game |
| `original_weapon_definitions` | `weapon_definition*` | static | Backup of default weapon definitions (for XML reset) |
| `original_shell_casing_definitions` | `shell_casing_definition*` | static | Backup of default shell casing definitions (for XML reset) |
| `original_weapon_ordering_array` | `int16*` | static | Backup of default weapon ordering (for XML reset) |
| `ShellCasingParser` | `XML_ShellCasingParser` | static | XML parser instance for shell casings |
| `WeaponOrderParser` | `XML_WeaponOrderParser` | static | XML parser instance for weapon order |
| `WeaponsParser` | `XML_ElementParser` | static | Parent XML parser for weapons system |

## Key Functions / Methods

### initialize_weapon_manager
- **Signature:** `void initialize_weapon_manager(void)`
- **Purpose:** One-time initialization of weapon system at engine startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `player_weapons_array` for all players; clears to zero
- **Calls:** `malloc()`, `objlist_clear()`
- **Notes:** Must be called before any player weapon initialization

### initialize_player_weapons_for_new_game
- **Signature:** `void initialize_player_weapons_for_new_game(short player_index)`
- **Purpose:** Reset weapon state for a player starting a new game
- **Inputs:** `player_index` ΓÇô player to initialize
- **Outputs/Return:** None
- **Side effects:** Clears all weapon data, initializes shell casings, resets light intensity
- **Calls:** `get_player_weapon_data()`, `obj_clear()`, `initialize_player_weapons()`, `initialize_shell_casings()`
- **Notes:** Called when creating a new player in new game

### update_player_weapons
- **Signature:** `void update_player_weapons(short player_index, uint32 action_flags)`
- **Purpose:** Main per-frame weapon update; drives state machine and animation
- **Inputs:** `player_index`, `action_flags` (trigger and movement input)
- **Outputs/Return:** None
- **Side effects:** Updates weapon state, animation sequences, firing, reloading, weapon luminosity; may fire projectiles
- **Calls:** `update_shell_casings()`, `handle_trigger_down()`, `handle_trigger_up()`, `update_sequence()`, `fire_weapon()`, `reload_weapon()`, many internal helpers
- **Notes:** Called once per frame from game loop; complex state machine with 16+ weapon states

### fire_weapon
- **Signature:** `static void fire_weapon(short player_index, short which_trigger, _fixed charged_amount, bool flail_wildly)`
- **Purpose:** Execute weapon firing: create projectile(s), play sound, update stats, apply recoil
- **Inputs:** `player_index`, `which_trigger` (primary/secondary), `charged_amount` (for charged weapons), `flail_wildly` (spray fire on death)
- **Outputs/Return:** None
- **Side effects:** Creates projectile(s), plays firing sound, updates shots_fired counter, modifies weapon intensity decay
- **Calls:** `calculate_weapon_origin_and_vector()`, `new_projectile()`, `play_weapon_sound()`, `FLIP_WEAPON_VARIANCE_SIGN()`, `get_shell_casing_definition()`
- **Notes:** Handles variance/spread, burst fire, guided missiles; may create shell casings

### update_shell_casings
- **Signature:** `static void update_shell_casings(short player_index)`
- **Purpose:** Update positions and velocities of all shell casing particles for a player
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Updates shell casing velocity (applies gravity/drag), removes casings that drift off-screen
- **Calls:** Direct array iteration; no external calls
- **Notes:** Simple particle physics; runs every frame

### new_shell_casing
- **Signature:** `static short new_shell_casing(short player_index, short type, uint16 flags)`
- **Purpose:** Allocate and initialize a shell casing particle
- **Inputs:** `player_index`, `type` (shell casing type), `flags` (e.g., reversed/flipped)
- **Outputs/Return:** Shell casing index or `NONE` if no free slots
- **Side effects:** Populates shell casing data array; marks slot as used
- **Calls:** `get_player_weapon_data()`, `get_shell_casing_definition()`
- **Notes:** Limited to `MAXIMUM_SHELL_CASINGS` (4) per player; includes randomized initial velocity variation

### reload_weapon
- **Signature:** `static bool reload_weapon(short player_index, short which_trigger)`
- **Purpose:** Initiate reload sequence for a trigger
- **Inputs:** `player_index`, `which_trigger`
- **Outputs/Return:** `true` if reload started; `false` if already reloading or no ammo available
- **Side effects:** Changes trigger state to awaiting/waiting reload; schedules reload timing
- **Calls:** `get_player_trigger_data()`, `player_weapon_has_ammo()`
- **Notes:** Handles both onscreen and offscreen reloading; coordinates with two-handed weapon logic

### ready_weapon
- **Signature:** `bool ready_weapon(short player_index, short weapon_index)` (not static, used by Pfhortran)
- **Purpose:** Bring a weapon into ready-to-fire state (raising animation)
- **Inputs:** `player_index`, `weapon_index`
- **Outputs/Return:** `true` if readying; `false` if weapon cannot be wielded
- **Side effects:** Sets weapon state to raising; schedules ready timing; auto-loads ammo if needed
- **Calls:** `get_player_weapon_data()`, `get_weapon_definition()`, `get_trigger_definition()`, `put_rounds_into_weapon()`
- **Notes:** Called when selecting a weapon; skipped if weapon has no weapon_class or fists are NONE

### get_weapon_display_information
- **Signature:** `bool get_weapon_display_information(short *count, struct weapon_display_information *data)`
- **Purpose:** Fetch per-frame weapon display data for HUD rendering (iterates over multiple calls)
- **Inputs:** `count` (iterator state, incremented), `data` (output buffer)
- **Outputs/Return:** `true` if valid display data populated; `false` if end of list
- **Side effects:** Populates `data` with shape, position, animation frame, transfer mode
- **Calls:** `get_current_weapon_definition()`, `get_shape_animation_data()`, `get_player_weapon_data()`, `update_sequence()` (via shell casings)
- **Notes:** Iterator pattern; handles idle animation, firing animation, shell casings, two-weapon positioning

### process_new_item_for_reloading
- **Signature:** `void process_new_item_for_reloading(short player_index, short item_type)`
- **Purpose:** Handle weapon/ammo pickup; reload applicable triggers
- **Inputs:** `player_index`, `item_type` (weapon or ammo)
- **Outputs/Return:** None
- **Side effects:** Resets trigger data, loads rounds, may switch to new weapon, updates ammo display
- **Calls:** `get_weapon_definition()`, `reset_trigger_data()`, `should_switch_to_weapon()`, `ready_weapon()`, routing based on item type/weapon class
- **Notes:** Large switch statement; handles random ammo, shared ammo between triggers, dual-pistol logic

### discharge_charged_weapons
- **Signature:** `void discharge_charged_weapons(short player_index)`
- **Purpose:** Fire all charged weapons when player dies
- **Inputs:** `player_index`
- **Outputs/Return:** None
- **Side effects:** Fires any weapons in charged state
- **Calls:** `get_player_weapon_data()`, `get_active_trigger_count_and_states()`, `fire_weapon()`
- **Notes:** Provides dramatic death animation where charged weapons discharge

### check_player_weapons_for_environment_change
- **Signature:** `void check_player_weapons_for_environment_change(void)`
- **Purpose:** Called on level entry; disables weapons that don't work in current environment
- **Inputs:** None (uses global `dynamic_world`)
- **Outputs/Return:** None
- **Side effects:** Switches player to different weapon if current doesn't work in environment; calls `calculate_ticks_from_shapes()`
- **Calls:** `dynamic_world` iteration, `weapon_works_in_current_environment()`, `select_next_best_weapon()`, `calculate_ticks_from_shapes()`
- **Notes:** Prevents environment-incompatible weapons (e.g., flamethrower in vacuum)

### Pack/Unpack Functions
- **Inline serialization helpers:** `StreamToTrigData()`, `TrigDataToStream()`, `StreamToWeapData()`, `WeapDataToStream()`, `StreamToShellData()`, `ShellDataToStream()`, `StreamToTrigDefData()`, `TrigDefDataToStream()`
- **Purpose:** Convert weapon/trigger/shell casing data to/from byte streams for save/network transmission
- **Calls:** `StreamToValue()`, `ValueToStream()` (macro-like functions from Packing.h)
- **Notes:** All data must be serialized in exact order for consistency; includes bounds checks

### XML Parsers
- **`XML_ShellCasingParser::HandleAttribute()`** ΓÇô Parse shell casing XML attributes (collection, shape, initial position/velocity, acceleration)
- **`XML_WeaponOrderParser::HandleAttribute()`** ΓÇô Parse weapon ordering (weapon selection priority)
- **`Weapons_GetParser()`** ΓÇô Return root XML parser; registers child parsers

## Control Flow Notes
This file fits into the **frame update** and **initialization** phases:
- **Init:** `initialize_weapon_manager()` at startup; `initialize_player_weapons_for_new_game()` when new player created; `check_player_weapons_for_environment_change()` on level entry
- **Frame update:** `update_player_weapons()` called every tick from main game loop, driving the weapon state machine
- **Display:** `get_weapon_display_information()` called during render phase to fetch HUD weapon sprites
- **Input-driven:** Weapon state changes triggered by action flags (trigger down/up, weapon switching) passed via `update_player_weapons()`

Weapon state machine has 16+ states: idle, raising, lowering, charging, charged, firing, recovering, reload variants, two-weapon variants. Transitions happen based on phase counters and external events (trigger input, ammo availability).

## External Dependencies
- **Notable includes:** `cseries.h`, `map.h`, `projectiles.h`, `player.h`, `weapons.h`, `SoundManager.h`, `interface.h`, `items.h`, `monsters.h`, `game_window.h`, `Packing.h`, `shell.h`, `weapon_definitions.h`, `XML_ElementParser.h`
- **External symbols (defined elsewhere):**
  - `weapon_definitions` (array), `shell_casing_definitions` (array), `weapon_ordering_array` (array) ΓÇô from `weapon_definitions.h`
  - `dynamic_world`, `current_player_index` ΓÇô from `map.h` / `player.h`
  - `get_player_data()`, `get_item_kind()` ΓÇô from `player.h`, `items.h`
  - `new_projectile()`, `mark_projectile_collections()` ΓÇô from `projectiles.h`
  - `SoundManager::instance()->PlaySound()` ΓÇô from `SoundManager.h`
  - `GetMemberWithBounds()`, `objlist_clear()` ΓÇô from `cseries.h` / memory utilities
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadBoundedNumericalValue()` ΓÇô XML/string parsing utilities
