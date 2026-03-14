# Source_Files/Network/network_dialog_widgets_sdl.cpp

## File Purpose
Implements network-related dialog widgets for the SDL UI system in Aleph One (Marathon engine). Provides custom widgets for displaying discovered network players, in-game player status with score/carnage graphs, and level selection for network games. Serves both the gather/join phase and postgame carnage report display.

## Core Responsibilities
- Display and manage lists of players discovered on the network
- Render player icons, names, and score/kill/death bars during and after games
- Handle mouse interactions with player elements (selection, graph clicks)
- Layout player names and bars to avoid overlapping text
- Provide level/entry-point selection for network games
- Convert between network topology data and display-ready player entries

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `w_found_players` | class (widget subclass) | List widget for discovered network players during gathering |
| `w_players_in_game2` | class (widget subclass) | Displays active players and postgame carnage reports with bars/icons |
| `w_entry_point_selector` | class (button subclass) | Level selector dropdown for network game setup |
| `player_entry2` | struct | Per-player display data (name, color, width, rendered image) |
| `bar_info` | struct | Score or carnage bar metadata (position, color, label) |
| `prospective_joiner_info` | struct | Network-discovered player info (name, color, team, gathering flag) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kWPIG2Width`, `kWPIG2Height` | enum constant | static | Widget dimensions (600├ù142) |
| `kMaxHeadroom`, `kNameOffset`, etc. | enum constants | static | Layout spacing for player icons and text |
| `postgame_layout` flag macros | macro | static | Conditional layout offsets for postgame vs. normal display |
| `operator==` for `prospective_joiner_info` | global function | file scope | Enables vector search by stream_id |
| `find_item_index_in_vector<T>()` | template function | file scope | Generic vector index lookup (comments note it's archaic) |

## Key Functions / Methods

### w_found_players::found_player
- **Signature:** `void found_player(prospective_joiner_info &player)`
- **Purpose:** Register a newly discovered network player and add to display list
- **Inputs:** Player info (name, color, team, stream_id)
- **Outputs/Return:** None; updates `found_players` and `listed_players` vectors, marks widget dirty
- **Side effects:** Calls `list_player()` to update display; increments `num_items`
- **Calls:** `list_player()`
- **Notes:** Assumes SSLP callbacks invoke this when players join

### w_found_players::hide_player
- **Signature:** `void hide_player(const prospective_joiner_info &player)`
- **Purpose:** Remove player from display (e.g., drop, cancel)
- **Inputs:** Player info to hide
- **Outputs/Return:** None
- **Side effects:** Calls `unlist_player()` to remove from display
- **Notes:** Adds to `hidden_players` but the comment suggests this is unused/legacy

### w_found_players::draw_item
- **Signature:** `void draw_item(vector<...>::const_iterator i, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- **Purpose:** Render a single player entry in the list
- **Inputs:** Iterator to player entry, SDL surface, position, width, selection state
- **Outputs/Return:** Text drawn to surface
- **Side effects:** Calls `draw_text()` with name, color, and "(gathering)" suffix if applicable
- **Calls:** `draw_text()`, `text_width()`, font/style queries
- **Notes:** Grays out players marked as `gathering`; centers text horizontally

### w_players_in_game2::update_display
- **Signature:** `void update_display(bool inFromDynamicWorld = false)`
- **Purpose:** Refresh player list from either network topology or in-game dynamic world
- **Inputs:** Boolean flag: true for postgame/dynamic world, false for topology
- **Outputs/Return:** None; updates `player_entries` and `players_on_team` arrays
- **Side effects:** Allocates `PlayerImage` objects per player; clears and rebuilds vectors; marks dirty
- **Calls:** `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, `get_player_data()`, `text_width()`, team/color lookups
- **Notes:** Allocates new `PlayerImage` for each player; name conversion differs for topology (pstring) vs. dynamic world (cstring)

### w_players_in_game2::draw
- **Signature:** `void draw(SDL_Surface* s) const`
- **Purpose:** Main rendering entry point for the widget
- **Inputs:** SDL surface
- **Outputs/Return:** Fully rendered widget to surface
- **Side effects:** Sets clip rectangle, draws all visual elements, resets clip
- **Calls:** Helper methods for icons (clumped/separate), bars (clumped/separate), names, labels, carnage totals, legend
- **Notes:** Controls z-order via draw sequence; uses `TextLayoutHelper` to avoid name/label overlaps; draws bars last so labels float upward

### w_players_in_game2::draw_player_icons_separately / draw_player_icons_clumped
- **Signature:** `void draw_player_icons_separately(SDL_Surface* s) const` / `draw_player_icons_clumped(SDL_Surface* s) const`
- **Purpose:** Position and render player 3D images; two strategies based on clump mode
- **Inputs:** SDL surface
- **Outputs/Return:** Icons drawn to surface
- **Calls:** `draw_player_icon()`, spacing calculations, `PlayerImage::drawAt()`
- **Notes:** Separate mode spreads players evenly across width; clumped mode groups by team then spreads within groups. Uses `selected_player` to dim unselected icons.

### w_players_in_game2::draw_bar_or_bars
- **Signature:** `void draw_bar_or_bars(SDL_Surface* s, size_t rank_index, int center_x, int maximum_value, vector<bar_info>& outBarInfos) const`
- **Purpose:** Draw score or carnage (kill/death/suicide) bars for a single player
- **Inputs:** Rank index, screen x-position, max bar value for scaling, output vector for labels
- **Outputs/Return:** Bar(s) drawn; bar metadata added to `outBarInfos` for later label placement
- **Side effects:** Calls `draw_bar()` 1ΓÇô2 times depending on mode; appends to `outBarInfos`
- **Calls:** `draw_bar()`, text formatting functions, string lookups
- **Notes:** Conditionally draws kills/deaths side-by-side or suicides for selected player; uses `kUseLegendThreshhold` to switch label format

### w_players_in_game2::draw_bar
- **Signature:** `void draw_bar(SDL_Surface* s, int inCenterX, int inBarColorIndex, int inBarValue, int inMaxValue, bar_info& outBarInfo) const`
- **Purpose:** Low-level bar rendering with 3D bevel effect
- **Inputs:** Surface, center x, bar color index, value to draw, max value for scale, output bar info
- **Outputs/Return:** Bar drawn to surface; bar metadata filled
- **Side effects:** Calls `SDL_FillRect()` 3 times (bright, middle, dark) for bevel effect
- **Calls:** `get_net_color()`, `SDL_MapRGB()`, `SDL_FillRect()`
- **Notes:** Handles negative and positive scales independently; supports 3-tone bevel effect

### w_entry_point_selector::validateEntryPoint
- **Signature:** `void validateEntryPoint()`
- **Purpose:** Ensure selected entry point is valid for current game type; fallback to first available
- **Inputs:** Uses `mGameType` and `mEntryPoint.level_number` as input state
- **Outputs/Return:** Updates `mEntryPoint` and `mCurrentIndex`; marks widget dirty
- **Side effects:** Clears and rebuilds `mEntryPoints` vector via network calls
- **Calls:** `get_entry_point_flags_for_game_type()`, `get_entry_points()`
- **Notes:** If current level not available for new game type, falls back to first entry point

### w_entry_point_selector::gotSelected / event
- **Signature:** `void gotSelected()` / `void event(SDL_Event &e)`
- **Purpose:** Handle UI interactionΓÇöpopup dialog for level selection or keyboard navigation
- **Inputs:** (gotSelected): none; (event): SDL_Event (keyboard SDLK_LEFT/RIGHT)
- **Outputs/Return:** (gotSelected): runs modal dialog, updates entry point on OK; (event): swallows arrow keys, cycles through entry points
- **Calls:** Dialog creation/layout, `play_dialog_sound()`
- **Notes:** Only shows dialog if multiple entry points exist; keyboard allows cycling without dialog

## Control Flow Notes
- **Gather/Join phase:** `w_found_players` lists discovered players; callbacks invoke `found_player()`, `update_player()`, `hide_player()` as network discovers/drops players.
- **In-game display:** `w_players_in_game2` renders via `update_display()` (from topology or dynamic world), then `draw()` is called each frame.
- **Postgame carnage report:** Same `w_players_in_game2` widget with `postgame_layout=true`, fed ranking data via `set_graph_data()`.
- **Level selection:** `w_entry_point_selector` sits in network dialogs; `event()` handles real-time keyboard cycling; `gotSelected()` spawns modal if user clicks.
- **Not inferable:** Whether these widgets are in the main game loop's `update_display()` or a separate dialog/UI event loop; likely the latter.

## External Dependencies
- **SDL graphics:** `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`, event types
- **Network:** `network.h` (topology, `NetGetNumberOfPlayers()`, `NetGetPlayerData()`), `SSLP_API.h` (service discovery), `prospective_joiner_info`
- **Game world:** `player.h` (player data, `MAXIMUM_NUMBER_OF_PLAYERS`, `NUMBER_OF_TEAM_COLORS`, `get_player_data()`), `player_info`, `entry_point`
- **UI framework:** `sdl_widgets.h` (`w_list<>`, `w_select_button`, widget base class), `dialog` class
- **Rendering:** `screen_drawing.h` (`draw_text()`, `text_width()`, theme colors), `sdl_fonts.h` (`font_info`), `HUDRenderer.h` (indirectly for color definitions)
- **Player visuals:** `PlayerImage_sdl.h` (`PlayerImage` class)
- **Text layout:** `TextLayoutHelper` (avoid overlapping text)
- **Strings:** `TextStrings.h` (`TS_GetCString()`), `network_dialogs.h` (`net_rank` structure)
- **Utilities:** `collection_definition.h`, `preferences.h`, `shell.h` (global sound/interface functions)

**Defined elsewhere:** `get_dialog_player_color()`, `calculate_ranking_text_for_post_game()`, `calculate_max_kills()`, `get_net_color()`, `dialog_cancel` callback, `clear_screen()`, `DIALOG_CLICK_SOUND`, etc.
