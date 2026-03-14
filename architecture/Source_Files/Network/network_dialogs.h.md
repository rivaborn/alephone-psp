# Source_Files/Network/network_dialogs.h

## File Purpose

Defines abstract dialog classes and interfaces for network game initialization (gathering players, joining, and setup), post-game statistics display, and metaserver integration. Provides a platform-agnostic interface implemented separately for Carbon (Mac) and SDL platforms.

## Core Responsibilities

- Define abstract factory classes for game dialogs (gather, join, setup) with platform-specific implementations
- Manage callback interfaces for network events during pre-game and chat phases
- Define data structures for ranking, game outcome tracking, and UI state
- Expose postgame carnage report rendering functions (graphs, rankings, damage stats)
- Provide enums and constants for dialog items, string resources, and game configuration limits

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `net_rank` | struct | Player ranking data: kills, deaths, friendly-fire kills, and score |
| `NetgameOutcomeData` | struct | UI control references for postgame report display |
| `GatherDialog` | class | Abstract base for server gathering players before game start |
| `JoinDialog` | class | Abstract base for clients joining a gatherer |
| `SetupNetgameDialog` | class | Abstract base for host configuring game parameters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rankings` | `struct net_rank[]` | global | Player ranking data used during postgame report rendering |

## Key Functions / Methods

### GatherDialog::Create
- **Signature:** `static std::auto_ptr<GatherDialog> Create()`
- **Purpose:** Abstract factory; returns platform-specific implementation
- **Return:** `std::auto_ptr<GatherDialog>` to gather dialog instance
- **Side effects:** Allocates and initializes dialog implementation

### GatherDialog::GatherNetworkGameByRunning
- **Signature:** `bool GatherNetworkGameByRunning()`
- **Purpose:** Runs the gather dialog until game starts or user cancels
- **Return:** `true` if successful, `false` if cancelled
- **Side effects:** Blocks until game gathering completes; manages player collection

### GatherDialog callbacks
- **JoinSucceeded, JoiningPlayerDropped, JoinedPlayerDropped, JoinedPlayerChanged**
- **Purpose:** Network callbacks when joiner status changes
- **Inputs:** `const prospective_joiner_info*` describing the joiner
- **Side effects:** Updates UI widget state reflecting gathered/ungathered player list

### JoinDialog::JoinNetworkGameByRunning
- **Signature:** `const int JoinNetworkGameByRunning()`
- **Purpose:** Runs join dialog; connects to gatherer and waits for game start
- **Return:** Player index on success, error code on failure
- **Side effects:** Blocks; performs network join announcements

### SetupNetgameDialog::SetupNetworkGameByRunning
- **Signature:** `bool SetupNetworkGameByRunning(player_info*, game_info*, bool, bool&)`
- **Purpose:** Runs setup dialog; collects all game parameters from host
- **Inputs:** Player data, game settings, resuming flag, metaserver advertising output flag
- **Return:** `true` if user confirmed, `false` if cancelled
- **Side effects:** Populates game_info and player_info structures

### Postgame report functions (external)
- **find_graph_mode, draw_new_graph, draw_player_graph, calculate_rankings, draw_totals_graph, draw_team_total_scores_graph**
- **Purpose:** Render postgame carnage reports with player statistics, team scores, kill/death graphs
- **Calls:** `get_net_color()`, sorting comparators (`rank_compare`, `team_rank_compare`, `score_rank_compare`)

## Control Flow Notes

Network game lifecycle:
1. **Gathering phase** (GatherDialog): Server waits for joiners; callbacks fire as players arrive
2. **Joining phase** (JoinDialog): Clients search and connect to gatherer; chat can occur
3. **Setup phase** (SetupNetgameDialog): Host configures map, game type, limits (only if gathering succeeded)
4. **In-game**: Game runs via network module
5. **Postgame**: Carnage report rendering via `display_net_game_stats()` and associated draw functions

Chat can occur during both gather and join phases; callbacks invoke `sendChat()` and `chatTextEntered()`.

## External Dependencies

- **Notable includes:**  
  `player.h` (MAXIMUM_NUMBER_OF_PLAYERS), `network.h` (game_info, player_info, prospective_joiner_info, ChatCallbacks, GatherCallbacks), `network_private.h` (JoinerSeekingGathererAnnouncer), `FileHandler.h` (FileSpecifier, FileChooserWidget), `network_metaserver.h` (metaserver client), `metaserver_dialogs.h` (GlobalMetaserverChatNotificationAdapter), `shared_widgets.h` (widget base classes: ButtonWidget, EditTextWidget, SelectorWidget, ToggleWidget, PlayersInGameWidget, etc.)

- **STL:** `<map>` (player tracking), `<set>` (game protocol), `<string>`, `<memory>` (auto_ptr)

- **Defined elsewhere:** `prospective_joiner_info`, `player_info`, `game_info` (from network.h), CFStringRef and OSType (Carbon/Mac specifics for NIB-based dialogs)
