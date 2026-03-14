# Source_Files/Network/network_dialogs.cpp

## File Purpose

Implements cross-platform (SDL/macOS) UI dialogs for network game setup. Handles gathering (hosting), joining, and configuring multiplayer games, with integration for metaserver advertising, LAN discovery via SSLP, and in-game/pre-game chat.

## Core Responsibilities

- **Network Gather Dialog**: UI for hosting a game, accepting joining players, starting game
- **Network Join Dialog**: UI for finding and connecting to gathering games via LAN or metaserver
- **Game Setup Dialog**: Configure game options (map, difficulty, game type, time/kill limits, player appearance)
- **Metaserver Integration**: Advertise hosted games and discover Internet games via metaserver
- **LAN Discovery (SSLP)**: Find/announce games on local network using kNetworkSetupProtocolID
- **Chat System**: Pre-game chat between gathered/joining players and metaserver chat
- **Progress Dialogs**: Display loading/setup progress during game initialization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `GatherDialog` | class | Base gather-dialog controller; manages players, chat, game start |
| `JoinDialog` | class | Base join-dialog controller; handles connection attempts and chat |
| `SetupNetgameDialog` | class | Base setup-dialog for game configuration (map, options, limits) |
| `SdlGatherDialog` | class | SDL-specific gather dialog implementation with widgets |
| `SdlJoinDialog` | class | SDL-specific join dialog implementation with tabbed UI |
| `SdlSetupNetgameDialog` | class | SDL-specific setup dialog with left/right layout |
| `GathererAvailableAnnouncer` | class | Registers game service on LAN via SSLP |
| `JoinerSeekingGathererAnnouncer` | class | Discovers games on LAN via SSLP callbacks |
| `game_info` | struct | Game config: map, difficulty, time/kill limits, options (from network.h) |
| `player_info` | struct | Player data: name, color, team (from network.h) |
| `prospective_joiner_info` | struct | Candidate player: stream_id, name, color, team, gathering flag |
| `ChatHistory` | class | Stores chat messages with sender info (defined elsewhere) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverClient` | `MetaserverClient*` | global | Singleton connection to metaserver; manages chat/game advertising |
| `gMetaserverChatHistory` | `ChatHistory` | global | Metaserver chat messages displayed in dialogs |
| `gPregameChatHistory` | `ChatHistory` | global | Pre-game (LAN) chat messages |
| `sProgressDialog` | `dialog*` | static | Progress dialog for loading/setup (non-SDL) |
| `sProgressMessage` | `w_static_text*` | static | Message widget in progress dialog |
| `sProgressBar` | `w_progress_bar*` | static | Progress bar widget |

## Key Functions / Methods

### network_gather
- **Signature:** `bool network_gather(bool inResumingGame)`
- **Purpose:** Entry point for hosting a network game. Sets up game/player info, enters network, creates GatherDialog, optionally advertises on metaserver.
- **Inputs:** `inResumingGame` ΓÇö whether resuming a saved game
- **Outputs/Return:** `true` if gather succeeded and game started; `false` if cancelled
- **Side effects:** Creates/deletes `gMetaserverClient`, calls `NetEnter()`, `NetGather()`, `NetDoneGathering()` or `NetCancelGather()`
- **Calls:** `network_game_setup()`, `NetEnter()`, `NetGather()`, `GatherDialog::Create()`, `GathererAvailableAnnouncer` constructor/destructor, `GameAvailableMetaserverAnnouncer`, `NetRetargetJoinAttempts()`, metaserver exception handlers
- **Notes:** Handles metaserver login exceptions; manages auto-ptr for metaserver announcer

### network_join
- **Signature:** `int network_join(void)`
- **Purpose:** Entry point for joining a network game. Manages join dialog flow and post-join metaserver updates.
- **Inputs:** None
- **Outputs/Return:** `kNetworkJoinedNewGame`, `kNetworkJoinedResumeGame`, `kNetworkJoinFailedJoined`, or `kNetworkJoinFailedUnjoined`
- **Side effects:** Calls `NetEnter()`, `JoinDialog::Create()->JoinNetworkGameByRunning()`, updates preferences, updates metaserver mode
- **Calls:** `NetEnter()`, `JoinDialog::Create()`, `NetGetGameData()`, `write_preferences()`, `read_preferences()`, `NetCancelJoin()`, `NetExit()`
- **Notes:** Saves preferences on successful join; reverts on failure

### GatherDialog::GatherNetworkGameByRunning
- **Signature:** `bool GatherNetworkGameByRunning()`
- **Purpose:** Run gather dialog event loop until game starts or user cancels.
- **Inputs:** None (uses member state)
- **Outputs/Return:** `true` if game started; `false` if cancelled
- **Side effects:** Sets chat callbacks, clears chat histories, binds autogather preference, activates/deactivates start button
- **Calls:** `Run()`, `binder.migrate_*()`, `write_preferences()`, chat setup functions
- **Notes:** Updates start button enabled state based on metaserver connection and player count

### GatherDialog::idle
- **Signature:** `void idle()`
- **Purpose:** Per-frame update called during dialog event loop. Handles player search and auto-gather.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `m_ungathered_players` map, calls network polling functions
- **Calls:** `MetaserverClient::pumpAll()`, `player_search()`, `NetGetNumberOfPlayers()`, `gathered_player()`
- **Notes:** Auto-gathers if preference enabled and space available

### GatherDialog::gathered_player
- **Signature:** `bool gathered_player(const prospective_joiner_info& player)`
- **Purpose:** Accept a prospective joiner into the game.
- **Inputs:** `player` ΓÇö joiner info
- **Outputs/Return:** `true` if gather succeeded; `false` if failed or at max players
- **Side effects:** Removes from `m_ungathered_players`, updates UI widget, calls `NetGatherPlayer()`
- **Calls:** `NetGetNumberOfPlayers()`, `NetGatherPlayer()`, `update_ungathered_widget()`
- **Notes:** Returns `false` silently if at player limit

### JoinDialog::JoinNetworkGameByRunning
- **Signature:** `const int JoinNetworkGameByRunning()`
- **Purpose:** Run join dialog event loop with preference binding.
- **Inputs:** None (uses member state)
- **Outputs/Return:** Join result code (kNetworkJoined* or kNetworkJoinFailed*)
- **Side effects:** Binds preferences (name, color, team, join address), runs dialog loop
- **Calls:** `binders.migrate_all_*()`, `Run()`, preference binding/migration functions
- **Notes:** Stores/restores preferences at start/end

### JoinDialog::attemptJoin
- **Signature:** `void attemptJoin()`
- **Purpose:** Initiate join attempt with optional address hint or LAN discovery.
- **Inputs:** None (reads widgets)
- **Outputs/Return:** None
- **Side effects:** Calls `NetGameJoin()`, disables input widgets, creates `JoinerSeekingGathererAnnouncer` if needed
- **Calls:** `NetGameJoin()`, `respondToJoinHit()`, widget enable/disable, `JoinerSeekingGathererAnnouncer` constructor
- **Notes:** Allocates hintString if join-by-address enabled; caller responsible for deletion after join attempt

### JoinDialog::gathererSearch
- **Signature:** `void gathererSearch()`
- **Purpose:** Check join state and update UI/result based on network events.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `join_result`, may call `Stop()`, updates team/color widgets, displays messages
- **Calls:** `JoinerSeekingGathererAnnouncer::pump()`, `MetaserverClient::pumpAll()`, `NetUpdateJoinState()`, various UI updates
- **Notes:** Handles state transitions (connecting ΓåÆ waiting ΓåÆ gathering ΓåÆ starting); one-time setup on first gather

### open_progress_dialog / close_progress_dialog / draw_progress_bar
- **Signature:** `void open_progress_dialog(size_t message_id, bool show_progress_bar)`; `void close_progress_dialog(void)`; `void draw_progress_bar(size_t sent, size_t total)`
- **Purpose:** Manage simple progress dialog for loading operations.
- **Inputs:** Message ID, optional progress bar flag; progress amounts
- **Outputs/Return:** None
- **Side effects:** Allocates/deallocates dialog and widgets; draws/updates progress
- **Calls:** Dialog and widget constructors/destructors; `sProgressDialog->start/finish/quit/draw()`
- **Notes:** Non-blocking; requires manual event processing calls

## Control Flow Notes

**Gather flow (host):**
1. User calls `network_gather()` ΓåÆ call `network_game_setup()` for game config
2. `NetEnter()` establishes network
3. `NetGather()` initializes gathering
4. `GatherDialog::GatherNetworkGameByRunning()` runs event loop with idle callback
5. Idle checks for joining players via `player_search()`, auto-gathers if enabled
6. User clicks START or all players ready ΓåÆ `NetDoneGathering()` + metaserver updates

**Join flow (client):**
1. User calls `network_join()` ΓåÆ creates `JoinDialog`
2. `JoinDialog::JoinNetworkGameByRunning()` binds preferences and runs dialog
3. User enters address/name/options ΓåÆ `attemptJoin()` calls `NetGameJoin()`
4. `gathererSearch()` callback checks `NetUpdateJoinState()` each frame
5. On gather success ΓåÆ update metaserver, return result code

**Metaserver/LAN discovery:**
- `GathererAvailableAnnouncer` registers via `SSLP_Allow_Service_Discovery()` during gather
- `JoinerSeekingGathererAnnouncer` searches via `SSLP_Locate_Service_Instances()` during join
- Callbacks update `NetRetargetJoinAttempts()` with discovered addresses
- Metaserver parallel: `gMetaserverClient` connects/advertises/receives chat asynchronously

## External Dependencies

- **Includes:** `network.h`, `network_private.h`, `SSLP_API.h`, `metaserver_dialogs.h`, `TextStrings.h`, `network_dialog_widgets_sdl.h`, `SoundManager.h`, `progress.h`
- **Network layer** (defined elsewhere): `NetEnter()`, `NetGather()`, `NetGameJoin()`, `NetCheckForNewJoiner()`, `NetUpdateJoinState()`, `NetGetGameData()`, `NetGetNumberOfPlayers()`, `NetChangeColors()`, `NetSetGatherCallbacks()`, `NetSetChatCallbacks()`, `SendChatMessage()`
- **Metaserver**: `MetaserverClient` class (singleton, manages chat & game advertising)
- **LAN discovery (SSLP)**: `SSLP_Allow_Service_Discovery()`, `SSLP_Disallow_Service_Discovery()`, `SSLP_Pump()`, `SSLP_Locate_Service_Instances()`, `SSLP_Stop_Locating_Service_Instances()` (external C functions)
- **Preferences**: `network_preferences`, `player_preferences`, `serial_preferences` (global pointers)
- **UI/Dialogs (SDL)**: `dialog`, `w_button`, `w_toggle`, `w_text_entry`, `w_select_popup`, `vertical_placer`, `horizontal_placer`, `table_placer`, `tab_placer` (widget framework, defined elsewhere)
- **Widget adapters**: `ButtonWidget`, `EditTextWidget`, `ToggleWidget`, `PopupSelectorWidget`, `ColorfulChatWidget`, `JoiningPlayerListWidget`, `PlayersInGameWidget`, etc. (defined in `network_dialog_widgets_sdl.h`)
