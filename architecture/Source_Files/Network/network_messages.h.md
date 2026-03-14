# Source_Files/Network/network_messages.h

## File Purpose
Defines network message types and classes used for Aleph One multiplayer game setup and initialization. Implements serializable message classes that communicate player info, capabilities, topology, map/physics/script data, and chat over the network using a stream-based encoding/decoding pattern.

## Core Responsibilities
- Define message type IDs for all network protocol messages (kHELLO_MESSAGE, kJOINER_INFO_MESSAGE, etc.)
- Implement typed message classes that inherit from SmallMessageHelper or BigChunkOfDataMessage base classes
- Provide serialization (deflate) and deserialization (inflate) via AIStream/AOStream
- Support compressed variants of large data messages (zipped map, physics, Lua)
- Manage client state machine and message dispatch handlers during game gather/join
- Provide template utilities for creating strongly-typed message classes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TemplatizedSimpleMessage<tMessageType, tValueType>` | template class | Generic wrapper for simple value-typed messages with clone/type support |
| `AcceptJoinMessage` | class | Communicates accept/reject decision with NetPlayer data |
| `CapabilitiesMessage` | class | Exchanges protocol capability versions (gameworld, star, ring, Lua, etc.) |
| `ChangeColorsMessage` | class | Updates player color/team selection |
| `ClientInfoMessage` | class | Adds/updates/removes client chat info with stream ID and action code |
| `HelloMessage` | class | Initial handshake with version string |
| `JoinerInfoMessage` | class | Joiner's initial info (prospective_joiner_info struct + version) |
| `BigChunkOfZippedDataMessage` | class | Wraps BigChunkOfDataMessage with automatic gzip compression on deflate, decompression on inflate |
| `TemplatizedDataMessage<messageType, T>` | template class | Templatized variant of T (BigChunkOfDataMessage or BigChunkOfZippedDataMessage) with specific message type ID |
| `NetworkChatMessage` | class | Chat text with sender ID, target (players/team/individual), max 1024 chars |
| `ServerWarningMessage` | class | Server-to-client warnings (e.g., "joiner ungatherable") |
| `TopologyMessage` | class | Network topology (player list, identifiers, game state) |
| `Client` | struct | Client state machine tracking connection state, capabilities, message handlers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Message type enum | enum (kHELLO_MESSAGE=700, etc.) | file | Protocol message type constants |
| `Client::check_player` | CheckPlayerProcPtr | static/struct | Callback function pointer for player validation |

## Key Functions / Methods

### AcceptJoinMessage::AcceptJoinMessage(bool accepted, NetPlayer *player)
- **Signature:** Constructor taking acceptance flag and player pointer
- **Purpose:** Construct message indicating whether server accepts joiner
- **Inputs:** `accepted` (bool), `player` (NetPlayer*)
- **Outputs/Return:** None (constructor)
- **Side effects:** Copies accepted flag and player struct into message members
- **Calls:** SmallMessageHelper constructor
- **Notes:** Player data is deep-copied; accessors provide pointer to internal copy

### CapabilitiesMessage::CapabilitiesMessage(const Capabilities &capabilities)
- **Signature:** Constructor taking Capabilities object
- **Purpose:** Package protocol capability versions for handshake
- **Inputs:** `capabilities` (const Capabilities&)
- **Outputs/Return:** None (constructor)
- **Side effects:** Copies Capabilities (which is a map<string, uint32>)
- **Calls:** SmallMessageHelper constructor
- **Notes:** Used to negotiate feature support (Lua, zipped data, etc.)

### HelloMessage::HelloMessage(const std::string &version)
- **Signature:** Constructor taking version string
- **Purpose:** Create initial handshake message
- **Inputs:** `version` (const std::string&)
- **Outputs/Return:** None (constructor)
- **Side effects:** Stores version string for serialization
- **Calls:** SmallMessageHelper constructor
- **Notes:** First message exchanged in game setup; version enables compatibility checks

### NetworkChatMessage::NetworkChatMessage(const char *chatText, int16 senderID, int16 target, int16 targetID)
- **Signature:** Constructor with text, sender, target type, and target ID
- **Purpose:** Create chat message for broadcast or targeted delivery
- **Inputs:** `chatText` (const char*, nullable), `senderID` (int16), `target` (int16: kTargetPlayers/Team/Player/Clients/Client), `targetID` (int16)
- **Outputs/Return:** None (constructor)
- **Side effects:** Copies up to 1023 chars of text; null-terminates
- **Calls:** (none)
- **Notes:** Uses strncpy with hardcoded CHAT_MESSAGE_SIZE=1024; null chatText defaults to empty string

### Client::capabilities_indicate_player_is_gatherable(bool warn_joiner)
- **Signature:** Method taking optional warning flag
- **Purpose:** Determine if client's capabilities meet server requirements for game
- **Inputs:** `warn_joiner` (bool: _dont_warn_joiner or _warn_joiner)
- **Outputs/Return:** bool (true if player can join)
- **Side effects:** May send warning message to joiner if gatherable check fails
- **Calls:** (implementation in .cpp, not visible)
- **Notes:** Implementation depends on external Capabilities map validation

### Client::can_pregame_chat()
- **Signature:** Inline method taking no arguments
- **Purpose:** Query whether client can send/receive chat before game starts
- **Inputs:** None
- **Outputs/Return:** bool
- **Side effects:** None (read-only state check)
- **Calls:** (none)
- **Notes:** Returns true if state is _connected, _connected_but_not_yet_shown, _ungatherable, _joiner_didnt_accept, _awaiting_accept_join, or _awaiting_map

### Client::handleJoinerInfoMessage(JoinerInfoMessage*, CommunicationsChannel*)
- **Signature:** Handler for incoming joiner info message
- **Purpose:** Process joiner's initial information (declared but impl. in .cpp)
- **Inputs:** JoinerInfoMessage*, CommunicationsChannel*
- **Outputs/Return:** void
- **Side effects:** Updates client state; may trigger validation
- **Calls:** (implementation not visible)
- **Notes:** Complementary handlers exist for Capabilities, AcceptJoin, Chat, ChangeColors

### Client::drop()
- **Signature:** Method taking no arguments
- **Purpose:** Forcibly disconnect and clean up client
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Closes CommunicationsChannel, transitions to _disconnect state
- **Calls:** (implementation in .cpp)
- **Notes:** Destructor also calls drop()

## Control Flow Notes
This file is part of the network initialization phase, executed before or during game gather (server) and join (client) operations:
1. **Initial Handshake:** HELLO messages exchanged to verify version compatibility
2. **Player Registration:** Joiner sends JOINER_INFO; server responds with ACCEPT_JOIN or rejection
3. **Capability Negotiation:** Both sides exchange CAPABILITIES to agree on feature support
4. **Topology Exchange:** Server sends TOPOLOGY with player list and identifiers
5. **Asset Distribution:** MAP, PHYSICS, LUA (optionally zipped) streamed to joiner
6. **Pre-Game Chat:** Players can chat after reaching certain states but before game starts
7. **State Machine:** Client struct state enum tracks progression through these phases; message handlers transition state or trigger warnings

The message definitions are agnostic to transport; they use AStream for encoding and rely on Message base-class deflate/inflate for integration with the stream protocol layer.

## External Dependencies
- **AStream.h** ΓÇö AIStream, AOStream (big/little-endian serialization)
- **Message.h** ΓÇö SmallMessageHelper, BigChunkOfDataMessage, SimpleMessage, DatalessMessage, UninflatedMessage, Message base class
- **network_capabilities.h** ΓÇö Capabilities class (map of string ΓåÆ uint32 feature versions)
- **network_private.h** ΓÇö NetPlayer, NetTopology, ClientChatInfo, CommunicationsChannel, MessageDispatcher, MessageHandler, prospective_joiner_info
- **SDL_net.h** ΓÇö Uint8, SDL network types
- **cseries.h** ΓÇö Standard library includes, int16/uint32 typedefs
