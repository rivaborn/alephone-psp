# Source_Files/Network/network_dialog_widgets_sdl.h

## File Purpose
Defines custom SDL widgets for network-related dialogs in Aleph One's networking UI. Provides specialized list widgets for discovering/displaying network players, a postgame carnage report display widget, and a level selector that adapts to game type constraints.

## Core Responsibilities
- **Player discovery list**: Manages SSLP callback integration to display available network players dynamically
- **Player roster display**: Shows players currently in a game for both gather/join phases and postgame carnage reports
- **Postgame graph rendering**: Renders kill/score bars, player icons, and statistics in multiple visual layouts
- **Level/entry point selection**: Validates and presents level options compatible with the current game type
- **Callback management**: Routes user interactions (clicks, selections) back to dialog code via function pointers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `w_found_players` | class | List widget tracking SSLP-discovered prospective joiners; manages visibility filtering |
| `w_players_in_game2` | class | Dual-purpose widget: shows roster during setup and renders postgame carnage graphs |
| `w_entry_point_selector` | class | Dropdown-style selector for levels, validates/resets entry point when game type or map changes |
| `player_entry2` | struct | Per-player rendering state: name, color, dimensions, and player image handle |

## Global / File-Static State
None defined in this file. External state referenced: `prospective_joiner_info` (from network.h), `entry_point` (from map.h), `net_rank` (from network_dialogs.h).

## Key Functions / Methods

### w_found_players::found_player
- Signature: `void found_player(prospective_joiner_info &player)`
- Purpose: Notify widget that a new player was discovered via SSLP
- Inputs: Reference to prospective joiner info
- Outputs/Return: None
- Side effects: Updates `found_players` vector; recomputes `listed_players` if player not hidden
- Calls: (visible in this file) `list_player()`, `unlist_player()`
- Notes: Callback invoked from SSLP layer (potentially from different thread)

### w_found_players::update_player
- Signature: `void update_player(prospective_joiner_info &player)`
- Purpose: Refresh state for a player already in the list (e.g., name/address change)
- Inputs: Updated player info
- Outputs/Return: None
- Side effects: Updates entry in `found_players` if present
- Calls: `unlist_player()`, `list_player()`
- Notes: Identity keyed by reference/instance, not name or address

### w_found_players::hide_player
- Signature: `void hide_player(const prospective_joiner_info &player)`
- Purpose: Exclude a player from the displayed list (e.g., already gathering)
- Inputs: Player info to hide
- Outputs/Return: None
- Side effects: Adds to `hidden_players`; removes from `listed_players` if present
- Calls: `unlist_player()`
- Notes: Does not remove from `found_players`; player can be unhidden

### w_found_players::item_selected
- Signature: `virtual void item_selected()`
- Purpose: Invoked by base list widget when user clicks a player
- Inputs: None (implicitly uses `selection` from base class)
- Outputs/Return: None
- Side effects: Invokes `player_selected_callback` if set
- Calls: `player_selected_callback()` (if non-null)
- Notes: Callback is expected to remove the item from the list (caller responsibility)

### w_players_in_game2::update_display
- Signature: `void update_display(bool inFromDynamicWorld = false)`
- Purpose: Refresh player list and graph data from current game state or postgame world
- Inputs: Flag indicating source (dynamic world for postgame, topology for in-game)
- Outputs/Return: None
- Side effects: Updates `player_entries` vector; recomputes bar metrics; marks widget dirty for redraw
- Calls: (not visible) Game world or topology inspection functions
- Notes: Required to be called at least once with valid topology before draw

### w_players_in_game2::draw
- Signature: `virtual void draw(SDL_Surface *s) const`
- Purpose: Render player roster and/or postgame carnage graphs
- Inputs: SDL surface to draw on
- Outputs/Return: None
- Side effects: Draws player icons, names, bars, and labels to surface
- Calls: `draw_player_icons_*()`, `draw_player_names_*()`, `draw_bars_*()`, `draw_carnage_legend()`, etc.
- Notes: Chooses layout (separate vs. clumped by team) based on `postgame_layout` flag; invokes `element_clicked_callback` on icon hit

### w_players_in_game2::set_graph_data
- Signature: `void set_graph_data(const net_rank* inRankings, int inNumRankings, int inSelectedPlayer, bool inClumpPlayersByTeam, bool inDrawScoresNotCarnage)`
- Purpose: Configure postgame graph display: player rankings, selection highlight, team grouping, and stat type (carnage vs. scores)
- Inputs: Rankings array, count, selected player index, team clumping flag, stat type flag
- Outputs/Return: None
- Side effects: Updates `net_rankings`, `clump_players_by_team`, `draw_scores_not_carnage`, `selected_player`; marks dirty
- Calls: None visible
- Notes: Called to prepare postgame carnage report display

### w_entry_point_selector::setGameType
- Signature: `void setGameType(size_t inGameType)`
- Purpose: Change the game type filter and re-validate the current entry point
- Inputs: New game type index
- Outputs/Return: None
- Side effects: Updates `mGameType`; calls `validateEntryPoint()` if type changed
- Calls: `validateEntryPoint()`
- Notes: Ensures entry point remains valid for new game type

### w_entry_point_selector::validateEntryPoint
- Signature: `void validateEntryPoint()`
- Purpose: (Private) Lookup entry point matching current level number and game type; fall back to first available if invalid
- Inputs: None (uses `mEntryPoint.level_number` and `mGameType`)
- Outputs/Return: None
- Side effects: Updates `mEntryPoint` and `mCurrentIndex` in `mEntryPoints` vector; marks selection button dirty
- Calls: (not visible) Map file query functions for available entry points
- Notes: Sets `mEntryPoint.level_number` to `NONE` if no entry points available; updates display string

### w_entry_point_selector::event
- Signature: `virtual void event(SDL_Event& e)`
- Purpose: Handle keyboard events (left/right arrow keys) to cycle through entry points
- Inputs: SDL event
- Outputs/Return: None
- Side effects: Increments/decrements `mCurrentIndex`; updates `mEntryPoint`; marks dirty
- Calls: `validateEntryPoint()` (indirectly via index change)
- Notes: Allows user to navigate entry points without opening a popup dialog

## Control Flow Notes
- **Network discovery phase**: SSLP callbacks invoke `w_found_players::found_player()` ΓåÆ list is displayed ΓåÆ user clicks ΓåÆ `player_selected_callback()` fires
- **Postgame phase**: Game populates `net_rank` array ΓåÆ `set_graph_data()` called ΓåÆ `draw()` renders bars and icons
- **Level selection**: Game type change ΓåÆ `setGameType()` ΓåÆ `validateEntryPoint()` ΓåÆ display updates; user can also cycle via arrow keys
- These widgets integrate with the SDL dialog system (inherit from `w_list<>` or `w_select_button`); widget lifecycle tied to dialog lifetime.

## External Dependencies
- **Includes**: `sdl_widgets.h` (base widget classes), `SSLP_API.h` (service discovery), `player.h` (player constants like `MAXIMUM_PLAYER_NAME_LENGTH`), `PlayerImage_sdl.h` (player rendering), `network_dialogs.h` (ranking structs, game type info)
- **External symbols**: `prospective_joiner_info` (network.h), `entry_point` (map.h), `net_rank` (network_dialogs.h), `TextLayoutHelper` (not defined here), `bar_info` (not defined here)
- **Base classes**: `w_list<T>` (template from sdl_widgets.h), `w_select_button` (sdl_widgets.h), `widget` (sdl_widgets.h)
