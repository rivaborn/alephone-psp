# Source_Files/Network/network_games.h

## File Purpose
Declares the network game management interface for multiplayer games. Provides functions to initialize and update network game state, calculate player/team rankings, query game conditions (over/balls/scores), and retrieve UI text for rankings and network compass navigation.

## Core Responsibilities
- Initialize and update networked multiplayer game state each frame
- Calculate and rank players/teams by kills/deaths
- Format ranking data for UI display (in-game and post-game)
- Query game mode capabilities (has scores, has balls/flags)
- Track player-versus-player kills
- Determine game-over conditions
- Provide network compass state for each player

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_ranking_data` | struct | Pairs a player index with their ranking score |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters` | extern int32[NUMBER_OF_TEAM_COLORS][2] | global | Per-team game parameters (likely team-specific scores or settings) |

## Key Functions / Methods

### initialize_net_game
- Signature: `void initialize_net_game(void)`
- Purpose: Set up network game state at game start
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes global game state
- Calls: Not inferable from this file

### update_net_game
- Signature: `bool update_net_game(void)`
- Purpose: Update game state once per frame; check if game is over
- Inputs: None
- Outputs/Return: true if game is over, false otherwise
- Side effects: Updates rankings, game state
- Calls: Not inferable from this file

### get_player_net_ranking / get_team_net_ranking
- Signature: `long get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over)` and team variant
- Purpose: Query current ranking for a player or team
- Inputs: player/team index, game_is_over flag
- Outputs/Return: Ranking value; kills/deaths written to output pointers
- Side effects: None (query-only)
- Calls: Not inferable from this file

### calculate_player_rankings
- Signature: `void calculate_player_rankings(struct player_ranking_data *rankings)`
- Purpose: Compute sorted ranking array for all players
- Inputs: Output buffer for rankings
- Outputs/Return: None (fills rankings array)
- Side effects: Allocates or modifies rankings array
- Calls: Not inferable from this file

### calculate_ranking_text / calculate_ranking_text_for_post_game
- Signature: `void calculate_ranking_text(char *buffer, long ranking)` and post-game variant
- Purpose: Format a ranking value into human-readable text for UI
- Inputs: ranking score; output buffer
- Outputs/Return: None (writes to buffer)
- Side effects: String buffer modification
- Calls: Not inferable from this file

### player_killed_player
- Signature: `bool player_killed_player(short dead_player_index, short aggressor_player_index)`
- Purpose: Record a player kill event; update rankings
- Inputs: Dead player index, killer index
- Outputs/Return: true if kill was valid/recorded
- Side effects: Updates player rankings/scores
- Calls: Not inferable from this file

### game_is_over
- Signature: `bool game_is_over(void)`
- Purpose: Check if game end condition is met
- Inputs: None
- Outputs/Return: true if game should end
- Side effects: None (query-only)
- Calls: Not inferable from this file

### get_network_compass_state
- Signature: `short get_network_compass_state(short player_index)`
- Purpose: Retrieve compass state (cardinal directions + beacon flag) for player radar/minimap
- Inputs: Player index
- Outputs/Return: Bitmask from compass enum (NW/NE/SW/SE + use_beacon)
- Side effects: None (query-only)
- Calls: Not inferable from this file

**Trivial helpers** summarized:
- `current_net_game_has_scores()`, `current_game_has_balls()` ΓÇö Boolean mode queries
- `get_network_joined_message()` ΓÇö Format join announcement
- `get_entry_point_flags_for_game_type()` ΓÇö Retrieve spawn flags for game type

## Control Flow Notes
Fits into a frame-based loop: `initialize_net_game()` runs at game start; `update_net_game()` is called each frame to advance state; query functions (`get_player_net_ranking()`, `game_is_over()`, `get_network_compass_state()`) are called during gameplay and UI rendering. Ranking calculations are likely triggered periodically or on player kill events.

## External Dependencies
- `NUMBER_OF_TEAM_COLORS` ΓÇö constant defined elsewhere (likely game mode / balance config)
- No explicit #include directives visible, suggesting minimal local coupling
