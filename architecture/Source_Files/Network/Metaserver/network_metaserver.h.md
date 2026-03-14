# Source_Files/Network/Metaserver/network_metaserver.h

## File Purpose
Defines the metaserver client for Aleph One, a networked game engine. Manages player connections to the metaserver, room selection, chat/messaging, game announcements, and synchronized lists of players and games.

## Core Responsibilities
- Establish and maintain TCP connection to metaserver with authentication
- Synchronize and maintain lists of rooms, players, and games via incremental updates (add/delete/refresh verbs)
- Send and receive chat messages, private messages, and broadcast announcements
- Announce game state changes (creation, start, reset, deletion, player counts)
- Support notification callbacks via NotificationAdapter interface for UI/game integration
- Track local player properties (name, team, away status, targeting) and sync with server
- Manage player ignore lists and game compatibility checking

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| MetaserverMaintainedList\<T\> | template class | Generic synchronized list using add/delete/refresh verbs; maintains element map with optional target tracking |
| MetaserverClient | class | Main client; owns network channel, message dispatchers, and state (rooms, players, games) |
| NotificationAdapter | inner interface | Abstract callback interface for receiving room/chat/game/player updates |
| NotificationAdapterInstaller | inner RAII class | Scoped temporary installer for notification adapters |
| LoginDeniedException | exception class | Thrown on login failure with error code (BadUserOrPassword, RoomFull, AccountLocked, etc.) |
| ServerConnectException | exception class | Thrown when TCP connection to metaserver fails |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| s_instances | std::set\<MetaserverClient*\> | static member (MetaserverClient) | Registry of all active clients; used by pumpAll() to dispatch messages to all instances |
| s_ignoreNames | std::set\<std::string\> | static member (MetaserverClient) | Set of player names added to ignore list |

## Key Functions / Methods

### connect
- Signature: `void connect(const std::string& serverName, uint16 port, const std::string& userName, const std::string& userPassword)`
- Purpose: Establish TCP connection to metaserver and authenticate with username/password
- Inputs: Server hostname, port, user credentials
- Outputs/Return: None (throws on failure)
- Side effects: Creates CommunicationsChannel, MessageInflater, MessageDispatchers; registers in s_instances; initiates login handshake
- Calls: Not inferable from header
- Notes: Throws ServerConnectException or LoginDeniedException; initiates asynchronous handshake (completion via pump())

### disconnect
- Signature: `void disconnect()`
- Purpose: Close connection to metaserver and clean up resources
- Outputs/Return: None
- Side effects: Closes m_channel; deregisters from s_instances; notifies adapter via roomDisconnected()
- Notes: Safe to call if already disconnected

### pump
- Signature: `void pump()`
- Purpose: Poll and process one batch of pending incoming messages for this client
- Outputs/Return: None
- Side effects: Deserializes messages, dispatches to handlers, updates internal state, invokes NotificationAdapter callbacks
- Calls: Various handleXxxMessage() private methods
- Notes: Must be called regularly in main loop; non-blocking

### pumpAll
- Signature: `static void pumpAll()`
- Purpose: Pump all active MetaserverClient instances in one call
- Side effects: Calls pump() on each instance in s_instances
- Notes: Convenience for global message processing

### setPlayerName / playerName
- Signature: `void setPlayerName(const std::string& name)` / `const std::string& playerName() const`
- Purpose: Set/get local player's display name
- Side effects: setPlayerName sends NameAndTeamMessage to server
- Notes: playerName() returns m_playerName

### setAway / setMode / setPlayerTeamName
- Purpose: Announce player status changes to server (away flag, mode bits, team name)
- Inputs: Bool/string/uint16 status values
- Side effects: Send UpdatePlayerData messages; update local m_playerName, m_teamName, m_notificationAdapter state
- Notes: Away status includes optional message

### rooms / setRoom
- Signature: `const Rooms& rooms() const` / `void setRoom(const RoomDescription& room)`
- Purpose: Query available rooms or join a specific room
- Outputs: Rooms vector or void; setRoom sends RoomLoginMessage
- Notes: Room list maintained from server RoomListMessage updates

### playersInRoom / gamesInRoom
- Signature: `const std::vector<MetaserverPlayerInfo> playersInRoom() const` / `const std::vector<GameListMessage::GameListEntry> gamesInRoom() const`
- Purpose: Retrieve current players/games in joined room
- Outputs: Vector of entries extracted from m_playersInRoom/m_gamesInRoom MetaserverMaintainedList
- Notes: Read-only snapshots; updated via pump() from server messages

### sendChatMessage / sendPrivateMessage
- Signature: `void sendChatMessage(const std::string& message)` / `void sendPrivateMessage(MetaserverPlayerInfo::IDType destination, const std::string& message)`
- Purpose: Send public or private chat
- Inputs: Message text (and recipient ID for private)
- Side effects: Creates and sends ChatMessage or PrivateMessage to server
- Notes: Relayed by server to recipients

### announceGame / announcePlayersInGame / announceGameStarted / announceGameReset / announceGameDeleted
- Purpose: Announce game lifecycle events to metaserver (creation, player count changes, start, reset, deletion)
- Inputs: Game port/description (for announceGame), player count, time in seconds, or none
- Side effects: Send corresponding CreateGameMessage, StartGameMessage, ResetGameMessage, RemoveGameMessage; update m_gameDescription, m_gamePort, m_gameAnnounced
- Notes: Together form a protocol for syncing game state to server

### ignore / is_ignored
- Signature: `void ignore(const std::string& name)` / `void ignore(MetaserverPlayerInfo::IDType id)` / `bool is_ignored(MetaserverPlayerInfo::IDType id)`
- Purpose: Add player to/check membership in ignore list
- Side effects: ignore() adds to s_ignoreNames; is_ignored() checks membership
- Notes: Prevents private messages and chat from ignored players

### player_target / player_target / find_player / game_target / game_target / find_game
- Purpose: Manage and query "targeted" (selected) player/game for UI highlighting
- Inputs: ID or none
- Outputs: ID or pointer to MetaserverPlayerInfo / GameListEntry
- Side effects: Updates target flag in m_playersInRoom/m_gamesInRoom entries
- Notes: Delegates to underlying MetaserverMaintainedList; target() setters update old/new targets

### associateNotificationAdapter / notificationAdapter
- Signature: `void associateNotificationAdapter(NotificationAdapter* adapter)` / `NotificationAdapter* notificationAdapter() const`
- Purpose: Install or retrieve the current event listener
- Side effects: associateNotificationAdapter stores pointer in m_notificationAdapter
- Notes: Used by handlers to invoke callbacks (playersInRoomChanged, gamesInRoomChanged, receivedChatMessage, etc.)

## Control Flow Notes

**Initialization**: Application calls `connect(host, port, user, pass)` ΓåÆ initiates TCP connection and login handshake (async).

**Main Loop**: Application repeatedly calls `pump()` or `pumpAll()` (e.g., every frame) ΓåÆ deserializes incoming messages, dispatches to handler methods (e.g., handlePlayerListMessage), updates internal lists/state, and invokes NotificationAdapter callbacks (e.g., playersInRoomChanged).

**State Sync**: Server sends incremental list updates (PlayerListMessage, GameListMessage) with verb (add/delete/refresh); MetaserverMaintainedList applies these atomically.

**Targeting**: UI can set player_target() or game_target() to highlight a specific entry; target flag stored and propagated via adapter callbacks.

**Shutdown**: Application calls `disconnect()` ΓåÆ closes channel, notifies adapter via roomDisconnected().

**Static Dispatch**: pumpAll() iterates s_instances for bulk message processing across all clients.

## External Dependencies

**Notable includes:**
- `metaserver_messages.h` ΓÇö Message type definitions (RoomDescription, PlayerListMessage, GameListMessage, ChatMessage, etc.)
- `<exception>, <vector>, <map>, <memory>, <set>` ΓÇö STL containers and memory management
- `Logging.h` ΓÇö Logging macros (logAnomaly1, etc.)

**Forward declarations / defined elsewhere:**
- CommunicationsChannel ΓÇö TCP communication abstraction
- MessageInflater ΓÇö Message deserialization/decompression
- MessageDispatcher, MessageHandler ΓÇö Message routing and handling base classes
- Message, ChatMessage, PrivateMessage, BroadcastMessage ΓÇö Message types (defined in metaserver_messages.h)
