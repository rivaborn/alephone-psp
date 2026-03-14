# Source_Files/Network/network_games.cpp

## File Purpose
Implements multiplayer network game mode logic for the Aleph One engine. Manages game-specific rules, scoring mechanics, player rankings, compass navigation targets, and game-over conditions across nine distinct netgame types (CTF, King of the Hill, tag, rugby, etc.).

## Core Responsibilities
- **Game initialization** per netgame type (beacon placement, state setup)
- **Per-frame game rule updates** (scoring, ball/flag handling, time tracking)
- **Player and team ranking calculations** with type-specific scoring formulas
- **Game-over condition evaluation** (time/kill/capture limits, Lua overrides)
- **Compass navigation** (directional indicators to objectives/targets)
- **Player event handling** (kill consequences, team assignment)
- **UI text generation** (ranking displays, game mode labels, post-game summaries)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_ranking_data` (from header) | struct | Player index + ranking score pair for sorting |
| `player_data` | struct (external) | Per-player state including `netgame_parameters[2]` score array |
| `game_data` | struct (external) | Game configuration (type, options, kill_limit, difficulty) |
| `dynamic_data` | struct (external) | Global state (player_count, game_beacon location, game_player_index) |
| Compass state enums | enum | Bitmask flags for directions (`_network_compass_nw/ne/sw/se`) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters` | `int32[NUMBER_OF_TEAM_COLORS][2]` | global | Team-level game parameters (king-of-hill time, flag pulls, points, etc.) indexed by mode-specific enum values |
| Parameter index enums | `enum` | file-static | Constants like `_king_of_hill_time`, `_ball_carrier_time`, `_offender_time_in_base` for indexing parameter arrays |
| `NETWORK_COMPASS_SLOP` | `#define` | file-static | Angular tolerance (SIXTEENTH_CIRCLE) for compass directional detection |

## Key Functions / Methods

### initialize_net_game
- **Signature:** `void initialize_net_game(void)`
- **Purpose:** Set up game state for current netgame type at game start
- **Inputs:** None (reads `GET_GAME_TYPE()` from dynamic_world)
- **Outputs/Return:** None; modifies global state
- **Side effects:** Zeros `team_netgame_parameters`; sets `game_beacon` for hill-based modes (KOTH/defense); initializes `game_player_index` to NONE for ball/tag games
- **Calls:** `GET_GAME_TYPE()`, `get_polygon_data()`, iterates `map_polygons`
- **Notes:** Computes hill beacon by averaging polygon centers; handles cases where no hill exists (sets beacon to 0,0)

### get_player_net_ranking
- **Signature:** `long get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Calculate a player's ranking/score according to current game type rules
- **Inputs:** Player index; output pointers for kill/death counts; game completion flag
- **Outputs/Return:** Ranking as long; populates `*kills` and `*deaths`
- **Side effects:** None
- **Calls:** `get_player_data()`, `GET_GAME_TYPE()`, `GetLuaScoringMode()` for custom modes
- **Notes:** Nine-case switch covering kill_monsters, cooperative, CTF, KOTH, tag, defense, rugby, custom, and kill-man-with-ball. Defense mode contains commented work-in-progress. Lua custom modes support points/time ranking variants.

### get_team_net_ranking
- **Signature:** `long get_team_net_ranking(short team, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Team-level parallel to `get_player_net_ranking`
- **Inputs:** Team index; output pointers
- **Outputs/Return:** Team ranking as long
- **Side effects:** None
- **Calls:** Accesses `team_netgame_parameters`, `team_damage_given/taken`, `team_monster_damage_*`
- **Notes:** Mirrors player function structure but aggregates across team members

### update_net_game
- **Signature:** `bool update_net_game(void)`
- **Purpose:** Advance game state each frame (increment scoring timers, handle ball/flag logic)
- **Inputs:** None (reads global state)
- **Outputs/Return:** Bool (documented as game_is_over flag, but always returns falseΓÇöactual game-over check is in `game_is_over()`)
- **Side effects:** Increments player/team `netgame_parameters`; destroys balls when scored; updates `game_player_index`; calls `mark_player_network_stats_as_dirty()`
- **Calls:** `player_has_ball()`, `find_player_ball_color()`, `destroy_players_ball()`, `get_polygon_data()`, `PLAYER_IS_DEAD()` macros
- **Notes:** Large per-game-type switch. CTF logic checks for flag pulls at bases. Rugby/kill-man-with-ball destroy ball on goal. Tag increments "it" time. Defense checks offense in hill. Contains commented-out game-over assignment code.

### get_network_compass_state
- **Signature:** `short get_network_compass_state(short player_index)`
- **Purpose:** Determine which compass cardinal directions to highlight for a player
- **Inputs:** Player index
- **Outputs/Return:** Short bitmask of compass flags (`_network_compass_nw` | `_network_compass_ne` | etc.)
- **Side effects:** None
- **Calls:** `get_player_data()`, `get_polygon_data()`, `player_has_ball()`, `get_object_data()`, `arctangent()`, `NORMALIZE_ANGLE()`
- **Notes:** Supports Lua-controlled compass (reads `use_lua_compass[]`, `lua_compass_*` arrays) or game-type-driven compass. Calculates player-to-beacon bearing; highlights cardinal direction(s) within `NETWORK_COMPASS_SLOP` tolerance. Game-type logic: KOTH/defense show hill beacon; tag shows "it" player; rugby/KMWB show ball carrier.

### player_killed_player
- **Signature:** `bool player_killed_player(short dead_player_index, short aggressor_player_index)`
- **Purpose:** Handle kill event consequences (e.g., tag mode target change)
- **Inputs:** Indices of killed and killing players
- **Outputs/Return:** BoolΓÇöwhether kill should be attributed for scoring
- **Side effects:** In tag mode, reassigns "it" to dead player if not already; marks stats dirty
- **Calls:** `GET_GAME_TYPE()`, `mark_player_network_stats_as_dirty()`
- **Notes:** Currently only tag mode implemented; other modes do nothing. Returns true (attribute kill) in all paths.

### game_is_over
- **Signature:** `bool game_is_over(void)`
- **Purpose:** Evaluate whether game end conditions have been satisfied
- **Inputs:** None (reads global game state)
- **Outputs/Return:** Bool
- **Side effects:** May modify `game_information.game_options` and `game_time_remaining` to trigger end-game sequence with 2-second delay
- **Calls:** `GetLuaGameEndCondition()`, `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()`, `get_player_data()`, `GET_GAME_PARAMETER()`
- **Notes:** Priority order: time expired ΓåÆ Lua conditions ΓåÆ kill-limit checks. Kill-limit checks vary by type: CTF/custom check team flag_pulls array; rugby sums player points; defense checks max offender time. Gives 2-second buffer before true game-over for death animation visibility.

### calculate_player_rankings
- **Signature:** `void calculate_player_rankings(struct player_ranking_data *rankings)`
- **Purpose:** Sort all players by ranking score (highest first)
- **Inputs:** Pointer to output array
- **Outputs/Return:** None; fills output array
- **Side effects:** None
- **Calls:** `get_player_net_ranking()`, reads `dynamic_world->player_count`
- **Notes:** Simple selection sort; fills array with sorted player_ranking_data entries

### calculate_ranking_text / calculate_ranking_text_for_post_game
- **Signature:** `void calculate_ranking_text(char *buffer, long ranking)` / `void calculate_ranking_text_for_post_game(char *buffer, long ranking)`
- **Purpose:** Format ranking value as human-readable string (numeric, percentage, or MM:SS time) based on game type
- **Inputs:** Output buffer; ranking value
- **Outputs/Return:** None; writes to buffer via sprintf
- **Side effects:** sprintf writes to buffer; post-game version loads format strings from resources
- **Calls:** `GET_GAME_TYPE()`, `GetLuaScoringMode()`, `sprintf()`, `ABS()`, `getcstr()`
- **Notes:** Different formats per game type (numeric for kills/captures; percentage for coop; time for KOTH/tag/defense/KMWB). Post-game variant localizes via resource strings.

### get_network_score_text_for_postgame / current_net_game_has_scores / current_game_has_balls
- **Signature:** `bool get_network_score_text_for_postgame(char *buffer, bool team_mode)` / `bool current_net_game_has_scores(void)` / `bool current_game_has_balls(void)`
- **Purpose:** Utility predicates and text generators for game type properties
- **Inputs/Outputs:** Game type checks and string generation; simple predicates
- **Side effects:** sprintf in text generator
- **Calls:** `GET_GAME_TYPE()`, `getcstr()`, `sprintf()`
- **Notes:** Query functions return true for scored games (CTF, KOTH, rugby, etc.) and false for unscored (kill_monsters, coop). Ball-game check returns true for CTF, rugby, KMWB only.

### get_network_joined_message / get_entry_point_flags_for_game_type
- **Signature:** `void get_network_joined_message(char *buffer, short game_type)` / `long get_entry_point_flags_for_game_type(size_t game_type)`
- **Purpose:** Generate join message or map entry-point flags for a game type
- **Inputs:** Output buffer + game type / game type
- **Outputs/Return:** String or long flags bitmask
- **Side effects:** sprintf writes to buffer
- **Calls:** `getcstr()`, game-type switch statement
- **Notes:** Entry point flags map game types to valid player-start-location bits (e.g., `_capture_the_flag_entry_point`, `_rugby_entry_point`)

### player_has_ball (static helper)
- **Signature:** `static bool player_has_ball(short player_index, short color)`
- **Purpose:** Check if a player carries a ball of given color
- **Inputs:** Player index, ball color
- **Outputs/Return:** Bool
- **Side effects:** None
- **Calls:** `get_player_data()`
- **Notes:** Simple inventory check: `player->items[BALL_ITEM_BASE+color] > 0`

## Control Flow Notes
This file participates in the main game loop as follows:
- **Initialization**: `initialize_net_game()` called once at game start
- **Per-tick updates**: `update_net_game()` increments scoring and ball state each frame
- **Game-over check**: `game_is_over()` evaluated to determine when to end match
- **Event handlers**: `player_killed_player()` called on death (tag mode reassignment)
- **UI rendering**: `get_player_net_ranking()`, `calculate_ranking_text()`, `get_network_compass_state()` called for HUD/scoreboard display
- **Level loading**: `get_entry_point_flags_for_game_type()` used to filter spawn points by mode

The file is heavily driven by the `GET_GAME_TYPE()` macro, which reads a global game-type enum to branch all logic. Lua scripting integration allows custom scoring modes and game-end conditions.

## External Dependencies
- **cseries.h** ΓÇö base utilities (assert, sprintf, csprintf, vhalt)
- **map.h** ΓÇö map geometry (polygon_data, map_polygons), game_data, dynamic_data, macros (GET_GAME_TYPE, GET_GAME_OPTIONS, GET_GAME_PARAMETER)
- **items.h** ΓÇö item constants (BALL_ITEM_BASE), find_player_ball_color()
- **lua_script.h** ΓÇö GetLuaScoringMode(), GetLuaGameEndCondition(), lua compass control
- **player.h** ΓÇö player_data, damage_record, PLAYER_IS_DEAD, PLAYER_IS_TOTALLY_DEAD macros
- **network.h** ΓÇö network subsystem declarations
- **network_games.h** ΓÇö own header (function prototypes, compass enums)
- **game_window.h** ΓÇö mark_player_network_stats_as_dirty()
- **SoundManager.h** ΓÇö (included but unused in visible code)

**Defined-elsewhere symbols**: `dynamic_world`, `current_player_index`, `current_player`, `temporary`, `map_polygons`, `get_player_data()`, `get_polygon_data()`, `get_object_data()`, `destroy_players_ball()`, `arctangent()`, `NORMALIZE_ANGLE()`, `PLAYER_IS_DEAD()`, `PLAYER_IS_TOTALLY_DEAD()`
