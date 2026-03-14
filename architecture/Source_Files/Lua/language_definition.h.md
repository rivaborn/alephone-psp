# Source_Files/Lua/language_definition.h

## File Purpose

A symbolic constant mapping file for game scripting language (Pfhortran), translating human-readable mnemonic names to internal numeric IDs. Serves as a data definition file for items, monsters, damage types, sounds, projectiles, and game mechanics. Supports dual naming conventions (with and without `_` prefix) for backwards compatibility.

## Core Responsibilities

- Define item type mnemonics (weapons, powerups, ammo) and their numeric codes
- Define monster/enemy type mnemonics and classifications
- Define damage type constants for tracking source of harm
- Define player and monster action/mode states
- Define visual effect (fader) types for cinematics and screen transitions
- Define audio/sound event IDs for gameplay events
- Define projectile types and their associated numeric IDs
- Define polygon/level geometry type constants
- Provide dual naming interface for script compatibility

## Key Types / Data Structures

None (data mapping fileΓÇöno struct/class/typedef definitions).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Item mnemonics | `{"string", 0xNN}` pairs | Global (script symbols) | Map item names to type IDs (0x00ΓÇô0x23) |
| Monster mnemonics | `{"string", 0xNN}` pairs | Global (script symbols) | Map enemy types to IDs (0x00ΓÇô0x2E) |
| Damage types | `{"string", 0xNN}` pairs | Global (script symbols) | Map damage source labels to IDs (0x00ΓÇô0x17) |
| Monster classes | `{"string", 0xNN}` pairs | Global (script symbols) | Bitflag classifications for monster grouping (0x0001ΓÇô0x8000) |
| Player/monster actions | `{"string", 0xNN}` pairs | Global (script symbols) | State IDs for player movement/combat and monster behavior |
| Sound IDs | `{"string", NN}` pairs | Global (script symbols) | Map audio events to numeric IDs (0ΓÇô214) |
| Projectile types | `{"string", 0xNN}` pairs | Global (script symbols) | Map projectile classes to IDs (0x00ΓÇô0x26) |

## Key Functions / Methods

None (data mapping fileΓÇöno executable functions).

## Control Flow Notes

**Not applicable.** This is a declarative data file used at parse/load time by a script interpreter or preprocessor. Likely loaded during engine initialization to populate a symbol table for the Lua/Pfhortran scripting language.

## External Dependencies

- **Undefined symbolic constants** (referenced but not defined in this file):
  - `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, `_panel_is_double_shield_refuel`, `_panel_is_triple_shield_refuel`
  - `_weapon_fist`, `_weapon_pistol`, `_weapon_plasma_pistol`, `_weapon_shotgun`, `_weapon_assault_rifle`, `_weapon_smg`, `_weapon_flamethrower`, `_weapon_missile_launcher`, `_weapon_alien_shotgun`, `_weapon_ball`
  - `_game_of_most_points`, `_game_of_most_time`, `_game_of_least_points`, `_game_of_least_time`
  - These are defined elsewhere and referenced here as numeric values for script constants.

- **Language/system**: Pfhortran (game scripting language, based on file header comments); used for Lua-based or similar scripting system
- **Backwards compatibility note**: File preserves old naming schemes (underscore-prefixed) alongside cleaner modern names to avoid breaking existing scripts
