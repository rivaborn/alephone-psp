# Source_Files/Network/Metaserver/metaserver_dialogs.cpp

## File Purpose
Implements the UI dialogs and notification handlers for the metaserver client in Aleph One. Enables players to discover games, browse player lists, participate in chat, and join online games through a central metaserver interface.

## Core Responsibilities
- **Game announcement**: Register and announce games to the metaserver with full game configuration
- **UI orchestration**: Manage modal dialog lifecycle, widget callbacks, and user interactions (game selection, player targeting, chat entry)
- **Chat integration**: Route metaserver chat messages (public, private, broadcast, local) into a unified chat history
- **Player/game list management**: Display and sort available players and games; update UI state based on selection
- **Game joining**: Extract server address/port and signal the main loop with join intent
- **Update checking**: Poll for available updates and notify the user before multiplayer play

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `GameAvailableMetaserverAnnouncer` | class | Announces a locally-hosted game to the metaserver; encapsulates game description and lifecycle |
| `GlobalMetaserverChatNotificationAdapter` | class | Implements `MetaserverClient::NotificationAdapter` to bridge metaserver chat events into `gMetaserverChatHistory` |
| `MetaserverClientUi` | class | Abstract base for the main metaserver dialog UI; owns widgets and handles player interactions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverChatHistory` | `ChatHistory` | global | Aggregates all chat messages (public, private, broadcast, local) for display |
| `gMetaserverClient` | `MetaserverClient*` | global | Singleton managing metaserver connection and state (players/games in room, network I/O) |
| `user_informed` | `bool` | static in `setupAndConnectClient()` | Prevents repeated update notifications across multiple calls |
| `first_check` | `bool` | static in `setupAndConnectClient()` | Tracks whether initial update check has completed |

## Key Functions / Methods

### run_network_metaserver_ui
- **Signature:** `const IPaddress run_network_metaserver_ui()`
- **Purpose:** High-level entry point to launch the metaserver UI modal dialog and return the selected game's join address.
- **Inputs:** None (uses global `gMetaserverClient`)
- **Outputs/Return:** `IPaddress` with `.host` and `.port` populated for the selected game; uninitialized if dialog cancelled
- **Side effects:** Creates and destroys a `MetaserverClientUi` instance; modifies `gMetaserverClient` state
- **Calls:** `MetaserverClientUi::Create()`, `GetJoinAddressByRunning()`
- **Notes:** One-shot design; returns invalid address if user cancels

### setupAndConnectClient
- **Signature:** `void setupAndConnectClient(MetaserverClient& client)`
- **Purpose:** Configure player name, check for updates (with timeout logic), and establish metaserver connection.
- **Inputs:** Reference to `MetaserverClient` to configure
- **Outputs/Return:** None; modifies `client` and may display dialogs
- **Side effects:** Calls `client.setPlayerName()`, `client.connect()`; may show update dialog; maintains static flags `user_informed` and `first_check`
- **Calls:** `pstrdup()`, `a1_p2cstr()`, `Update::instance()`, dialog creation (`open_progress_dialog()`, `close_progress_dialog()`)
- **Notes:** Update check waits 500ms silently, then up to 2.5s with progress dialog. Uses platform-specific file handling for netscript path.

### GameAvailableMetaserverAnnouncer::GameAvailableMetaserverAnnouncer
- **Signature:** `GameAvailableMetaserverAnnouncer(const game_info& info)`
- **Purpose:** Constructor; immediately announces a game to the metaserver with full scenario, map, physics, and game options.
- **Inputs:** `game_info` containing game type, time limit, difficulty, map name, game options, cheat flags, kill limit
- **Outputs/Return:** None; announces via `gMetaserverClient->announceGame()`
- **Side effects:** Calls `setupAndConnectClient()`, populates `GameDescription`, retrieves map/physics file names, calls `announceGame()`
- **Calls:** `setupAndConnectClient()`, `FileSpecifier::SetSpec()`, `FileSpecifier::GetName()`, `FileSpecifier::Exists()`, `gMetaserverClient->announceGame()`
- **Notes:** Filters time limits > 7 days as "untimed" (-1). Handles netscript path conditionally (Mac-specific code path). Formats Aleph One version string with platform.

### GameAvailableMetaserverAnnouncer::Start
- **Signature:** `void Start(int32 time_limit)`
- **Purpose:** Signal that the announced game has started; update metaserver with actual game duration in seconds.
- **Inputs:** `time_limit` in seconds
- **Outputs/Return:** None
- **Side effects:** Calls `gMetaserverClient->announceGameStarted()`
- **Calls:** `MetaserverClient::announceGameStarted()`
- **Notes:** Complementary to constructor; server expects `Start()` after game announcement.

### GlobalMetaserverChatNotificationAdapter::receivedChatMessage / receivedPrivateMessage / receivedBroadcastMessage / receivedLocalMessage / roomDisconnected
- **Signature:** `void received*Message(...)` / `void roomDisconnected()`
- **Purpose:** Notification callbacks from `MetaserverClient` that append messages to the shared chat history and play UI feedback sounds.
- **Inputs:** Sender name, sender ID, message text (varies by method)
- **Outputs/Return:** None
- **Side effects:** Constructs `ColoredChatEntry`, sets color from sender's player info, appends to `gMetaserverChatHistory`, calls `PlayInterfaceButtonSound()`
- **Calls:** `gMetaserverClient->find_player()`, `PlayInterfaceButtonSound()`, `ColoredChatHistory::append()`
- **Notes:** Private messages clear the player selection after sending. Different sound for different message types.

### MetaserverClientUi::GetJoinAddressByRunning
- **Signature:** `const IPaddress MetaserverClientUi::GetJoinAddressByRunning()`
- **Purpose:** Main event loop; connect to metaserver, register callbacks, run modal dialog, and return join address if a game was selected.
- **Inputs:** None; uses internal widget state and `gMetaserverClient`
- **Outputs/Return:** `IPaddress` (host/port) of selected game, or zero-initialized if cancelled
- **Side effects:** Asserts `!m_used` (one-shot); connects client, registers notification adapter, clears chat history, binds all widget callbacks, calls `Run()`, may call `Stop()`
- **Calls:** `setupAndConnectClient()`, `bind()`, `boost::bind()`, various widget callbacks, `Run()`, `handleCancel()`
- **Notes:** Invokes abstract `Run()` (platform-specific); captures `this` in closures. One-shot pattern prevents reuse.

### MetaserverClientUi::GameSelected / PlayerSelected
- **Signature:** `void GameSelected(GameListMessage::GameListEntry game)` / `void PlayerSelected(MetaserverPlayerInfo info)`
- **Purpose:** Handle selection of a game or player in the list; toggle target and update button states.
- **Inputs:** Game entry or player info
- **Outputs/Return:** None
- **Side effects:** Updates `game_target()` / `player_target()` on `gMetaserverClient`; re-sorts and re-displays widget; calls `UpdateGameButtons()` / `UpdatePlayerButtons()`. Double-click within 333ms triggers `JoinClicked()`
- **Calls:** `std::sort()`, `MetaserverClientUi::UpdateGameButtons()`, `UpdatePlayerButtons()`, `JoinClicked()`, `Scenario::instance()->IsCompatible()`
- **Notes:** Click toggles selection off; double-click on a non-running, compatible game joins immediately

### MetaserverClientUi::JoinGame
- **Signature:** `void JoinGame(const GameListMessage::GameListEntry& game)`
- **Purpose:** Extract IP and port from game entry, populate `m_joinAddress`, and signal dialog exit.
- **Inputs:** Game entry with IP address and port
- **Outputs/Return:** None
- **Side effects:** Memcopy IP, set port, calls `Stop()`
- **Calls:** `memcpy()`, `Stop()`
- **Notes:** Simple bridge from game selection to network layer; `Stop()` triggers dialog exit and return to caller.

### MetaserverClientUi::sendChat
- **Signature:** `void sendChat()`
- **Purpose:** Retrieve text from chat entry widget, route to private or public message, clear field and player selection.
- **Inputs:** None; reads from `m_chatEntryWidget`
- **Outputs/Return:** None
- **Side effects:** Calls `gMetaserverClient->sendPrivateMessage()` if player targeted, else `sendChatMessage()`; clears player target; re-sorts/refreshes player list widget; clears text field
- **Calls:** `EditTextWidget::get_text()`, `MetaserverClient::sendPrivateMessage()`, `sendChatMessage()`, `set_text()`
- **Notes:** Private messages auto-clear the player selection and refresh list to show state change.

### color_entry (helper)
- **Signature:** `static int color_entry(ColoredChatEntry& e, const MetaserverPlayerInfo *player)`
- **Purpose:** Populate RGB color fields in a chat entry from the sender's player info.
- **Inputs:** Reference to `ColoredChatEntry` and optional pointer to `MetaserverPlayerInfo`
- **Outputs/Return:** None (implicitly 0; function signature appears incomplete)
- **Side effects:** Modifies `e.color.{red,green,blue}`
- **Calls:** None (direct field access)
- **Notes:** Called by all message receipt handlers to colorize chat by player.

## Control Flow Notes
- **Initialization / Join Phase**: `run_network_metaserver_ui()` ΓåÆ `setupAndConnectClient()` ΓåÆ `GetJoinAddressByRunning()` ΓåÆ widget setup and callback binding
- **Update Loop**: `MetaserverClient::pump()` (called implicitly by `Run()` during modal) processes network events and invokes notification callbacks (e.g., `playersInRoomChanged`, `gamesInRoomChanged`, chat events)
- **User Interaction**: Click/input events trigger widget callbacks (e.g., `GameSelected()`, `sendChat()`, `MuteClicked()`), which modify client state and refresh UI
- **Exit**: `JoinClicked()` ΓåÆ `JoinGame()` ΓåÆ `Stop()` breaks the modal loop and returns join address to caller
- **Cancellation**: `handleCancel()` deletes and recreates `gMetaserverClient`, then calls `Stop()`

## External Dependencies
- **network_metaserver.h**: `MetaserverClient`, `MetaserverPlayerInfo`, `GameListMessage`, `GameDescription`, `NotificationAdapter`
- **metaserver_messages.h**: Message types and utilities
- **preferences.h**: `player_preferences`, `network_preferences`, `environment_preferences` globals
- **network_private.h**: `GAME_PORT` constant
- **alephversion.h**: `A1_DISPLAY_VERSION`, `A1_DISPLAY_PLATFORM` macros
- **SoundManager.h**: `PlayInterfaceButtonSound()` function
- **Update.h**: `Update::instance()`, update status checking
- **progress.h**: `open_progress_dialog()`, `close_progress_dialog()`
- **shared_widgets.h**: Widget classes (`PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`)
- **CSeries / cseries.h**: `pstrdup()`, `a1_p2cstr()`, macro utilities
- **Standard library**: `std::vector`, `std::sort`, `std::string`, `std::auto_ptr`, `boost::bind()`
- **SDL**: `SDL_GetTicks()` for timing
