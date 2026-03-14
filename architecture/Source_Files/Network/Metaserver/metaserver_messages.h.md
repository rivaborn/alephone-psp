# Source_Files/Network/Metaserver/metaserver_messages.h

## File Purpose
Defines message types and serializable data structures for metaserver client-server communication. Implements a protocol for login, player/game list sync, chat, and game management using the TCPMess message framework with binary serialization via AIStream/AOStream.

## Core Responsibilities
- Define message type constants for serverΓåöclient bidirectional communication (login, room/player/game lists, chat, authentication)
- Provide serializable message classes inheriting from `SmallMessageHelper` for deflation/inflation with binary streams
- Define aggregate data structures (`GameDescription`, `RoomDescription`, `MetaserverPlayerInfo`) for game state representation
- Support authentication flow (salt exchange, login success, handoff tokens)
- Provide game creation, removal, and state-update messaging
- Support player roster, chat, and private messaging
- Handle scenario/map compatibility checks and game filtering logic

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `HandoffToken` | typedef | 32-byte opaque token for room server authentication |
| `GameDescription` | struct | Complete game state: type, time/kill limits, difficulty, players, map, scenario, physics |
| `RoomDescription` | class | Room metadata: ID, player/game counts, type (normal/ranked/tournament), server address |
| `MetaserverPlayerInfo` | class | Player roster entry: ID, name, admin flags, ranking, colors, team, status (away bit) |
| `GameListMessage::GameListEntry` | nested struct | Game listing with ID, IP, port, verb, description, time remaining, host player; implements sorting/filtering by compatibility |
| `LoginAndPlayerInfoMessage` | class | ClientΓåÆServer: username, player name, team name |
| `SaltMessage` | class | ServerΓåÆClient: encryption type and 16-byte salt for password hashing |
| `RoomListMessage` | class | ServerΓåÆClient: vector of `RoomDescription` |
| `PlayerListMessage` | class | ServerΓåÆClient: vector of `MetaserverPlayerInfo` |
| `GameListMessage` | class | ServerΓåÆClient: vector of `GameListEntry` with live filtering |
| `ChatMessage`, `PrivateMessage` | classes | Bidirectional: sender ID/name, message text, flags (directed bit), RGB color |

## Global / File-Static State
None. All state is instance-based within message objects.

## Key Functions / Methods

### GameDescription::GameDescription (constructor)
- **Signature:** `GameDescription()` (default)
- **Purpose:** Initialize game descriptor with defaults (8 players max, "Untitled Game", no teams/closure, fetch current scenario/protocol/version from singleton)
- **Inputs:** None
- **Outputs/Return:** Initialized struct
- **Side effects:** Queries `Scenario::instance()` at construction
- **Calls:** `Scenario::instance()->GetID()`, `GetName()`, `GetVersion()`
- **Notes:** Hardcoded defaults; scenario data fetched lazily from global singleton

### LoginAndPlayerInfoMessage::reallyDeflateTo
- **Signature:** `void reallyDeflateTo(AOStream& thePacket) const;`
- **Purpose:** Serialize login credentials to outbound packet
- **Inputs:** `AOStream` output sink
- **Outputs/Return:** None (writes to stream)
- **Side effects:** Modifies stream position
- **Calls:** (not visible; implemented in .cpp)
- **Notes:** One-way message; `reallyInflateFrom()` asserts false (server never receives this)

### SaltMessage::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream);`
- **Purpose:** Deserialize encryption salt and type from server challenge
- **Inputs:** `AIStream` inbound data
- **Outputs/Return:** bool (success)
- **Side effects:** Populates `m_encryptionType` and `m_salt[16]`
- **Calls:** (not visible; implemented in .cpp)
- **Notes:** Server-only message; `reallyDeflateTo()` asserts false (client never sends)

### MetaserverPlayerInfo::MetaserverPlayerInfo (constructor)
- **Signature:** `MetaserverPlayerInfo(AIStream& fromStream);`
- **Purpose:** Deserialize player roster entry from metaserver broadcast
- **Inputs:** `AIStream` binary data
- **Outputs/Return:** Constructed player info
- **Side effects:** Populates all member fields (ID, name, colors, rank, admin flags, status)
- **Calls:** (not visible; implemented in .cpp)
- **Notes:** Only deserialization constructor; no default/copy shown in header

### MetaserverPlayerInfo::sort (static comparator)
- **Signature:** `static bool sort(const MetaserverPlayerInfo& a, const MetaserverPlayerInfo& b);`
- **Purpose:** Multi-key sort: admin (desc), then status (asc), then playerID (asc)
- **Inputs:** Two player info references
- **Outputs/Return:** true if `a` sorts before `b`
- **Side effects:** None
- **Calls:** None
- **Notes:** Bungie admins > regular admins > non-admins; within tier, non-away before away; then by ID

### GameListMessage::GameListEntry::minutes_remaining
- **Signature:** `int minutes_remaining() const;`
- **Purpose:** Calculate time remaining in game, accounting for elapsed wall-clock time
- **Inputs:** None (uses `m_timeRemaining` and `m_ticks`)
- **Outputs/Return:** Minutes remaining (ΓëÑ0) or -1 if unlimited
- **Side effects:** None
- **Calls:** `SDL_GetTicks()`
- **Notes:** Returns 0 if result < 0 (game expired); assumes monotonic ticks

### GameListMessage::GameListEntry::compatible
- **Signature:** `bool compatible() const;`
- **Purpose:** Check if local scenario supports this game's scenario
- **Inputs:** None
- **Outputs/Return:** true if compatible
- **Side effects:** None
- **Calls:** `Scenario::instance()->IsCompatible(m_description.m_scenarioID)`
- **Notes:** Enables client-side game filtering

### GameListMessage::GameListEntry::sort (static comparator)
- **Signature:** `static bool sort(const GameListEntry& a, const GameListEntry& b);`
- **Purpose:** Multi-tier sort: (compatible Γêº !running), (compatible Γêº running), !compatible; within tier, by ID
- **Inputs:** Two game list entries
- **Outputs/Return:** true if `a` sorts before `b`
- **Side effects:** None
- **Calls:** `compatible()`, `running()`
- **Notes:** Joinable games (compatible, not running) ranked highest; incompatible games lowest

### RoomDescription::read
- **Signature:** `void read(AIStream& inStream);`
- **Purpose:** Deserialize room metadata from stream
- **Inputs:** `AIStream` binary data
- **Outputs/Return:** None
- **Side effects:** Populates `m_id`, `m_playerCount`, `m_gameCount`, `m_type`, `m_address`
- **Calls:** (not visible; implemented in .cpp)
- **Notes:** Accessor methods provided for read-only access to deserialized fields

## Control Flow Notes
**Not inferable from this file.** This is a pure data-definition header. Message inflation/deflation is implemented in a corresponding `.cpp` file. The message types enum suggests a classic request-response flow:
- Client sends `kCLIENT_LOGIN` ΓåÆ Server sends `kSERVER_SALT` ΓåÆ Client resends `kCLIENT_LOGIN`
- Client queries ΓåÆ Server sends `kSERVER_ROOMLIST`, `kSERVER_GAMELIST`, `kSERVER_PLAYERLIST`
- Bidirectional: `kBOTH_CHAT`, `kBOTH_PRIVATE_MESSAGE`, `kBOTH_KEEP_ALIVE`

Likely integrated into a network event loop that deserializes incoming `UninflatedMessage` ΓåÆ type-dispatched inflate ΓåÆ callback.

## External Dependencies
- **Message.h:** `SmallMessageHelper`, `DatalessMessage<T>` base classes; `COVARIANT_RETURN` macro
- **AStream.h:** `AIStream`, `AOStream` for binary serialization (abstract; concrete BE/LE variants exist)
- **SDL_net.h:** `IPaddress` type
- **Scenario.h:** `Scenario::instance()` singleton for scenario ID/name/version and compatibility checks
- **network.h:** `kNetworkSetupProtocolID` constant
- **std library:** `<string>`, `<vector>`
- **Boost:** `boost::algorithm::to_lower_copy()` for case-insensitive string handling (imported via using declaration)
