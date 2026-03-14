# Source_Files/Network/Metaserver/network_metaserver.cpp

## File Purpose

Implementation of the MetaserverClient class, a network client for connecting to the Aleph One metaserver. Handles authentication, room login, player/game list synchronization, chat messaging, game announcement, and player management for multiplayer game discovery and coordination.

## Core Responsibilities

- **Authentication & Connection**: Login to metaserver with encryption, room selection and login, token-based session handling
- **Message Handler Setup**: Inflate/deflate protocol messages, dispatch incoming messages by type to appropriate handlers
- **Player & Game Management**: Maintain synchronized lists of players and games in the current room, support player targeting
- **Chat & Communication**: Process chat messages with built-in commands (.available, .who, .ignore, .games), send private messages, handle broadcasts
- **Game Lifecycle**: Announce games, update player counts, mark games as started/closed/deleted, sync game state with server
- **Ignore List Management**: Toggle player ignore status, filter messages from ignored players, prevent muting guests
- **Connection Lifecycle**: Periodic network pump to send/receive messages, detect disconnection, notify adapter on connection changes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MetaserverClient` | class | Main metaserver client instance; manages connection, messaging, and game state |
| `MetaserverMaintainedList<T>` | template class | Maintains synchronized list of players or games with add/delete/refresh semantics |
| `NotificationAdapter` | abstract class | Callback interface for UI notifications (chat, player list changes, disconnection) |
| `LoginDeniedException` | exception class | Thrown on login failure with error codes and messages |
| `ServerConnectException` | exception class | Thrown on connection/protocol errors |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_instances` | `set<MetaserverClient*>` | static | Tracks all active MetaserverClient instances for pumpAll() |
| `s_ignoreNames` | `set<string>` | static | Global ignore list shared across all client instances |
| `kKeyLength` | int (const) | static | Key length (16) for password encryption |

## Key Functions / Methods

### connect
- **Signature:** `void connect(const string& serverName, uint16 port, const string& userName, const string& userPassword)`
- **Purpose:** Establish encrypted connection to metaserver, authenticate, receive room list, connect to room server, and receive player ID
- **Inputs:** Server hostname/IP, port, username, password
- **Outputs/Return:** None; throws on failure
- **Side effects:** 
  - Modifies `m_channel` connection state
  - Sets `m_playerID` from room login response
  - Populates `m_rooms` from RoomListMessage
  - Switches message dispatcher from login to normal mode
- **Calls:** 
  - `CommunicationsChannel::connect()`, `disconnect()`, `enqueueOutgoingMessage()`, `receiveMessage()`, `receiveSpecificMessageOrThrow()`
  - `SaltMessage::encryptionType()`, `salt()`
  - Message handler via dispatcher
- **Notes:** 
  - Implements password encryption (plaintext or XOR-based)
  - Selects "Arrival" room if available, else uses first room
  - Commented-out debug IP address setup suggests development mode
  - Throws `ServerConnectException` or `LoginDeniedException`

### pump
- **Signature:** `void pump()`
- **Purpose:** Process one cycle of network I/O and dispatch incoming messages; detect and notify disconnection
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Calls message handlers for incoming messages
  - Sets `m_notifiedOfDisconnected` flag on first disconnect detection
  - Notifies adapter of room disconnection
- **Calls:** `CommunicationsChannel::pump()`, `dispatchIncomingMessages()`, `notificationAdapter->roomDisconnected()`
- **Notes:** Safe to call repeatedly; only notifies disconnection once per instance

### pumpAll
- **Signature:** `static void pumpAll()`
- **Purpose:** Pump all active MetaserverClient instances in a single call
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `pump()` on each instance in `s_instances`
- **Calls:** Uses STL `for_each` with `mem_fun`

### sendChatMessage
- **Signature:** `void sendChatMessage(const string& message)`
- **Purpose:** Send chat message or process built-in commands (.available, .who, .ignore, .games, .pork)
- **Inputs:** Message text (commands prefixed with `.`)
- **Outputs/Return:** None
- **Side effects:** 
  - Enqueues outgoing ChatMessage (unless command)
  - Calls adapter callback with formatted player list or game list (for commands)
  - Modifies `s_ignoreNames` via `ignore()` for .ignore command
- **Calls:** `playersInRoom()`, `gamesInRoom()`, `ignore()`, `notificationAdapter->receivedLocalMessage()`
- **Notes:** 
  - Commands are local; .available filters non-away players
  - .ignore without args lists ignored names; .ignore <name> toggles ignore
  - .pork easter egg returns "NO BACON FOR YOU"

### announceGame
- **Signature:** `void announceGame(uint16 gamePort, const GameDescription& description)`
- **Purpose:** Advertise a new game on the metaserver
- **Inputs:** Local game port, game description (type, map, difficulty, player count, etc.)
- **Outputs/Return:** None
- **Side effects:** 
  - Sets `m_gameDescription`, `m_gamePort`, `m_gameAnnounced = true`
  - Enqueues CreateGameMessage
- **Calls:** `CommunicationsChannel::enqueueOutgoingMessage()`

### announceGameStarted
- **Signature:** `void announceGameStarted(int32 gameTimeInSeconds)`
- **Purpose:** Mark announced game as running and closed (no new joins)
- **Inputs:** Game duration in seconds (INT32_MAX = unlimited)
- **Outputs/Return:** None
- **Side effects:** Sets `m_gameDescription.m_closed`, `m_gameDescription.m_running`; enqueues UpdateGame and StartGame messages
- **Calls:** `enqueueOutgoingMessage()`

### handleChatMessage / handlePrivateMessage
- **Signature:** `void handle{Chat|Private}Message(ChatMessage|PrivateMessage* message, CommunicationsChannel* inChannel)`
- **Purpose:** Process incoming chat/private messages; filter ignored players and muted guests
- **Inputs:** Message object, channel (unused)
- **Outputs/Return:** None
- **Side effects:** 
  - Calls adapter callbacks for valid messages
  - Filters messages from ignored players and guest accounts (if mute_metaserver_guests enabled)
  - Removes formatting codes and special prefix characters from sender name
- **Calls:** `remove_formatting()`, `boost::algorithm::starts_with()`, `notificationAdapter->receivedChatMessage/receivedPrivateMessage()`
- **Notes:** Both methods share nearly identical filtering logic (potential refactoring opportunity)

### ignore
- **Signature:** 
  - `void ignore(const string& name)` 
  - `void ignore(MetaserverPlayerInfo::IDType id)`
- **Purpose:** Toggle a player on the global ignore list; notify adapter
- **Inputs:** Player name (with possible formatting codes) or player ID
- **Outputs/Return:** None
- **Side effects:** 
  - Adds/removes player from `s_ignoreNames` (global static set)
  - Calls adapter with confirmation message
- **Calls:** `remove_formatting()`, `m_playersInRoom.find()`, `notificationAdapter->receivedLocalMessage()`
- **Notes:** Cleans formatting codes before adding to ignore list

### remove_formatting (static helper)
- **Signature:** `static string remove_formatting(const string& s)`
- **Purpose:** Strip chat formatting codes (|p, |b, |i, |l, |r, |c, |s) from a string
- **Inputs:** String with possible formatting markers
- **Outputs/Return:** String with markers removed
- **Side effects:** None
- **Calls:** `tolower()`, `style_code()`

## Control Flow Notes

**Initialization ΓåÆ Connection ΓåÆ Pump Loop:**
1. Constructor creates CommunicationsChannel, MessageInflater, MessageDispatcher, and message handlers
2. `connect()` performs multi-phase login: metaserver authentication, room selection, room login
3. Main loop calls `pump()` (or `pumpAll()` for all instances) to:
   - Service network I/O (send queued messages, receive data)
   - Dispatch received messages to type-specific handlers
   - Detect disconnection and notify adapter
4. Application initiates chat/game actions via `sendChatMessage()`, `announceGame()`, etc.
5. `disconnect()` or destructor closes connection and removes instance from global set

**Two Dispatcher Modes:**
- **Login phase:** `m_loginDispatcher` accepts only SetPlayerDataMessage
- **Post-login phase:** `m_dispatcher` handles all message types (broadcasts, chat, player lists, game lists, keep-alive)

## External Dependencies

- **Networking:** `CommunicationsChannel`, `MessageInflater`, `MessageDispatcher`, `MessageHandler` (TCPMess library)
- **Message Types:** `SaltMessage`, `AcceptMessage`, `DenialMessage`, `ChatMessage`, `PrivateMessage`, `PlayerListMessage`, `GameListMessage`, `RoomListMessage`, etc. (metaserver_messages.h)
- **Game Data:** `GameDescription`, `RoomDescription`, `MetaserverPlayerInfo` (metaserver_messages.h)
- **Utilities:** `Logging.h` (logAnomaly), `boost::algorithm::starts_with`, std containers (set, vector, map)
- **System:** SDL_net (`IPaddress`), standard C++ (string, iostream, algorithm, memory)
