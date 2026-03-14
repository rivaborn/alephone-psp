# Source_Files/Lua/lua_mnemonics.h

## File Purpose
Defines constant lookup tables that map human-readable string identifiers (mnemonics) to integer codes for Lua script integration. Enables game content to reference entities, damage types, effects, sounds, and other game constants by name rather than magic numbers.

## Core Responsibilities
- Provide string-to-integer mappings for 23 distinct game concept categories
- Support Lua scripting access to engine-level constants
- Define sentinel-terminated lookup arrays for efficient runtime lookups
- Centralize mnemonic definitions for game entities (monsters, items, weapons, effects) and mechanics (difficulty, game modes, control panels)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lang_def` | struct | Maps a C string (`name`) to a 32-bit integer (`value`); used as the element type for all mnemonic arrays |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Collection_Mnemonics` | `const lang_def[]` | global | Game entity collection types (weapons in hand, explosions, water, lava, player, etc.) |
| `Lua_ControlPanelClass_Mnemonics` | `const lang_def[]` | global | Control panel types (oxygen recharger, shield recharger, light switch, terminal, etc.) |
| `Lua_DamageType_Mnemonics` | `const lang_def[]` | global | Damage source types (explosion, staff, projectile, flame, lava, etc.) |
| `Lua_DifficultyType_Mnemonics` | `const lang_def[]` | global | Game difficulty levels (kindergarten, easy, normal, major damage, total carnage) |
| `Lua_EffectType_Mnemonics` | `const lang_def[]` | global | Visual/audio effect types (explosions, splashes, sparks, teleports, etc.ΓÇö73 entries) |
| `Lua_FadeType_Mnemonics` | `const lang_def[]` | global | Screen fade/transition effects (cinematic fades, colored overlays, static, etc.) |
| `Lua_GameType_Mnemonics` | `const lang_def[]` | global | Multiplayer game modes (capture the flag, king of the hill, rugby, tag, etc.) |
| `Lua_ScoringMode_Mnemonics` | `const lang_def[]` | global | Scoring criteria using symbolic constants (`_game_of_most_points`, etc.) |
| `Lua_ItemType_Mnemonics` | `const lang_def[]` | global | Pickupable items (weapons, ammo, health, powerups, balls, keys) |
| `Lua_MediaType_Mnemonics` | `const lang_def[]` | global | Environmental liquids (water, lava, goo, sewage, jjaro) |
| `Lua_MonsterClass_Mnemonics` | `const lang_def[]` | global | Monster/entity class flags (player, bob, defender, hunter, cyborg, etc.ΓÇöbitmask values) |
| `Lua_MonsterAction_Mnemonics` | `const lang_def[]` | global | Monster behavior states (stationary, moving, attacking, dying, teleporting) |
| `Lua_MonsterMode_Mnemonics` | `const lang_def[]` | global | Monster targeting/lock states (locked, losing lock, unlocked, running) |
| `Lua_MonsterType_Mnemonics` | `const lang_def[]` | global | Specific monster variant types (minor/major ticks, fighters, drones, yetis, etc.ΓÇö48 entries) |
| `Lua_OverlayColor_Mnemonics` | `const lang_def[]` | global | HUD overlay colors (green, white, red, cyan, yellow, blue) |
| `Lua_PlayerColor_Mnemonics` | `const lang_def[]` | global | Multiplayer player colors (slate, red, violet, yellow, white, orange, blue, green) |
| `Lua_PolygonType_Mnemonics` | `const lang_def[]` | global | Map polygon types/triggers (teleporter, hill, base, goal, ouch, glue, etc.) |
| `Lua_ProjectileType_Mnemonics` | `const lang_def[]` | global | Projectile/weapon discharge types (missile, grenade, bullets, bolts, etc.) |
| `Lua_SceneryType_Mnemonics` | `const lang_def[]` | global | Static decorative/interactive objects (lights, cylinders, bones, containers, etc.ΓÇö61 entries) |
| `Lua_SideType_Mnemonics` | `const lang_def[]` | global | Map surface side classifications (full, high, low, split) |
| `Lua_Sound_Mnemonics` | `const lang_def[]` | global | Sound effect identifiers (startup, weapons fire, footsteps, monster calls, ambient loopsΓÇö215 entries) |
| `Lua_TransferMode_Mnemonics` | `const lang_def[]` | global | Texture/sprite animation modes (pulsate, wobble, slide, wander) |
| `Lua_WeaponType_Mnemonics` | `const lang_def[]` | global | Available weapons (fist, pistol, assault rifle, missile launcher, alien weapon, etc.) |

## Key Functions / Methods
None. This file contains only data definitions.

## Control Flow Notes
Not applicableΓÇöpure static data. Arrays are likely consumed by an adjacent lookup function (not shown) that performs binary or linear search given a string key to retrieve the corresponding integer value for Lua API binding.

## External Dependencies
- **Type `int32`**: Custom typedef or platform header; defines mnemonic values
- **Symbolic constants** (e.g., `_game_of_most_points` in `Lua_ScoringMode_Mnemonics`): Defined elsewhere, likely in a game constants header
- **Presumed lookup infrastructure**: Some other translation unit must iterate/search these arrays to bind Lua table keys to C values

## Notes
- **Sentinel termination**: Every array ends with `{0, 0}` to signal end-of-table for iteration
- **Mnemonic categories**: Covers entity types (monsters, items, weapons), mechanics (damage, effects, difficulty), environment (media types, polygon types, scenery), UI (overlay colors, fade types), and audio (sounds)
- **Bitmask vs. enum values**: `Lua_MonsterClass_Mnemonics` uses powers of 2 (bitflags), while most others use sequential or sparse integer IDs
- **Context**: References to "bob," "fighter," "yeti," "pfhor," "juggernaut" suggest this is the **Aleph One engine** (Marathon-based) or similar
- **File mislabel**: Header comment says `LUA_OBJECTS.CPP` but file is `.h`; likely outdated comment
