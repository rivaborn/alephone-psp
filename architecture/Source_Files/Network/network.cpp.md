# Source_Files/Network/network.cpp

## File Purpose

Core multiplayer networking implementation for Aleph One game engine. Manages network topology, player joining/dropping, game state transitions, and distribution of map/game data across the network. Supports both Ring and Star game protocols with message-based communication using SDL_net and TCP.

## Core Responsibilities

- **Network initialization & teardown**: Set up DDP socket, topology structures, establish connections
- **Player lifecycle management**: Handle joining players, dropping disconnected players, topology updates
- **Game protocol coordination**: Delegate to RingGameProtocol or StarGameProtocol for frame/action handling
- **Map & game data distribution**: Stream map/physics/Lua script data from gatherer to joiners
- **Message routing**: Dispatch incoming network messages (chat, capabilities, topology, etc.) via MessageDispatcher
- **Join state machine**: Manage connecting ΓåÆ joining ΓåÆ waiting ΓåÆ active phases for joiners
- **Topology maintenance**: Track player positions, addresses, identifiers; propagate changes to all players
- **Feature negotiation**: Validate player capabilities (protocol support, compression, Lua, etc.)
- **Ignore lists & player muting**: Per-local-player ignore/mute management for chat and microphone

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Client` | class | Represents a remote player connected to gatherer; holds channel, state, capabilities, message handlers |
| `Capabilities` | struct | Bitmask of supported features (Ring, Star, Zipped data, Lua, etc.) with versions |
| `NetTopologyPtr` | typedef | Pointer to current network topology (players, game data, addresses) |
| `RingGameProtocol` | class | Ring-topology protocol (all-to-all) for action/flag distribution |
| `StarGameProtocol` | class | Star-topology protocol (hub-spoke) for latency-based action queuing |
| `NetDistributionInfo` | struct | Maps distribution type ID to lossy flag and handler procedure |
| `client_map_t` | typedef | `std::map<int, Client*>` ΓÇô maps stream IDs to connected client objects |
| `distribution_info_map_t` | typedef | `std::map<int16, NetDistributionInfo>` ΓÇô maps distribution types to handlers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ddpSocket` | short | static | DDP UDP socket number |
| `localPlayerIndex` | short | static | This machine's index in topology |
| `localPlayerIdentifier` | short | static | This machine's unique player ID |
| `topology` | NetTopologyPtr | static | Current network topology (all players, game state) |
| `sCurrentGameProtocol` | NetworkGameProtocol* | static | Active protocol (Ring or Star) |
| `sRingGameProtocol`, `sStarGameProtocol` | static objects | static | Protocol instances, selected as needed |
| `connections_to_clients` | client_map_t | static | Map of stream ID ΓåÆ Client object for all joiners/players |
| `client_chat_info` | client_chat_info_map_t | static | Map of stream ID ΓåÆ ClientChatInfo (name, color, team) |
| `connection_to_server` | CommunicationsChannel* | static | Joiner's TCP connection to gatherer (NULL if gatherer) |
| `netState` | short | static | Current network state (netUninitialized, netGathering, netJoining, netWaiting, netActive, etc.) |
| `distribution_info_map` | distribution_info_map_t | static | STL map of distribution type handlers |
| `next_stream_id` | int | static | Counter for assigning unique stream IDs to joiners |
| `deferred_script_data`, `deferred_script_length` | byte*, size_t | static | Buffered Lua netscript to distribute on level start |
| `do_netscript` | bool | static | Whether netscript is enabled for current game |
| `sIgnoredPlayers` | std::set<int> | static | Set of player indices ignored by local player |
| `sDisplayPings` | bool | static | Console toggle for showing network latency |

## Key Functions / Methods

### NetInitializeTopology
- **Signature**: `static void NetInitializeTopology(void *game_data, short game_data_size, void *player_data, short player_data_size)`
- **Purpose**: Initialize topology for a new network game; set local player as index 0, identifier 0; copy initial game and player data.
- **Inputs**: Game state struct, player state struct, and their sizes (for safety bounds checking).
- **Outputs/Return**: None (modifies global `topology`).
- **Side effects**: Allocates/initializes `topology` structure; sets `localPlayerIndex`, `localPlayerIdentifier`, `topology->player_count = 1`.
- **Calls**: `NetLocalAddrBlock()`, `memcpy()`.
- **Notes**: Called once at game start; assumes local player is always index 0. Uses void* to maintain separation from Marathon game code (weak type safety).

### NetDistributeTopology
- **Signature**: `static void NetDistributeTopology(short tag)`
- **Purpose**: Broadcast current topology to all connected players (except local player and those with identifier NONE).
- **Inputs**: `tag` ΓÇô reason for update (tagNEW_PLAYER, tagDROPPED_PLAYER, tagCHANGED_PLAYER, etc.).
- **Outputs/Return**: None (sends via channels).
- **Side effects**: Enqueues TopologyMessage on all remote player channels; increments topology's tag field.
- **Calls**: `CommunicationsChannel::enqueueOutgoingMessage()`.
- **Notes**: Only called in netGathering state. Skips local player and invalid identifiers.

### NetDistributeGameDataToAllPlayers
- **Signature**: `OSErr NetDistributeGameDataToAllPlayers(byte *wad_buffer, long wad_length, bool do_physics)`
- **Purpose**: Stream map (and optionally physics/Lua) from gatherer to all joiners, handling compression and multiple recipients.
- **Inputs**: WAD buffer (map data), length, physics flag.
- **Outputs/Return**: OSErr (noErr on success, 1 on timeout).
- **Side effects**: Sends PhysicsMessage, MapMessage, LuaMessage (optionally zipped variants); updates progress dialog; sets all clients to state `_ingame` on success.
- **Calls**: `get_network_physics_buffer()`, `process_network_physics_model()`, `CommunicationsChannel::multipleFlushOutgoingMessages()`, progress UI functions.
- **Notes**: Separates clients into zip-capable and zip-incapable groups; uses boost::bind to apply messages to all channels; timeout = topology->player_count * MAP_TRANSFER_TIME_OUT.

### NetReceiveGameData
- **Signature**: `byte *NetReceiveGameData(bool do_physics)`
- **Purpose**: Joiner waits for map/physics/Lua from server via blocking message receive with timeout.
- **Inputs**: `do_physics` ΓÇô whether to process physics model after reception.
- **Outputs/Return**: Pointer to map buffer (caller frees), or NULL if timeout/error.
- **Side effects**: Processes physics and Lua if received; updates progress dialog; frees local buffers on failure.
- **Calls**: `connection_to_server->receiveSpecificMessage<EndGameDataMessage>()`, `process_network_physics_model()`, `LoadLuaScript()`.
- **Notes**: Waits 60s with 30s keepalive; relies on message handlers to fill handlerMapBuffer, handlerPhysicsBuffer, handlerLuaBuffer as side effect.

### NetUpdateJoinState
- **Signature**: `short NetUpdateJoinState(void)`
- **Purpose**: State machine for joiner: attempt connection, await Hello, await gather, await game start, with retry logic and error handling.
- **Inputs**: None (reads global `netState`, `host_address`, connection pool).
- **Outputs/Return**: New state (netConnecting, netJoining, netWaiting, netActive, netJoinErrorOccurred, or NONE).
- **Side effects**: Creates/abandons NonblockingConnect via ConnectPool; calls `connection_to_server->pump()` and `dispatchIncomingMessages()`; alerts user on error.
- **Calls**: `ConnectPool::instance()->connect()`, `setMessageInflater()`, `setMessageHandler()`, `alert_user()`.
- **Notes**: Retries connection every 5 seconds; handles DNS resolution failures and connection timeouts; uses handlerState for sub-state machine; does not set global netState if returning netPlayerAdded, netPlayerDropped, etc.

### NetCheckForNewJoiner
- **Signature**: `bool NetCheckForNewJoiner(prospective_joiner_info &info)`
- **Purpose**: Gatherer checks for incoming joiner connections, pumps all established connections, and returns next joiner ready to gather.
- **Inputs**: Reference to info struct (filled on return).
- **Outputs/Return**: true if a new joiner is ready to gather, false otherwise.
- **Side effects**: Accepts new TCP connection, sends HelloMessage, pumps/dispatches all client channels, drops disconnected clients, transitions client state from `_connected_but_not_yet_shown` to `_connected`.
- **Calls**: `server->newIncomingConnection()`, `Client::Client()`, message handlers.
- **Notes**: Prunes disconnected clients; relies on client message handlers to transition states.

### Client::handleJoinerInfoMessage
- **Signature**: `void Client::handleJoinerInfoMessage(JoinerInfoMessage* joinerInfoMessage, CommunicationsChannel*)`
- **Purpose**: Gatherer-side handler for joiner's player name/color/team info; validate protocol version, send capabilities back.
- **Inputs**: JoinerInfoMessage containing name, color, team, protocol version.
- **Outputs/Return**: None (sends CapabilitiesMessage, updates client_chat_info, broadcasts ClientInfoMessage to other clients).
- **Side effects**: Sets client state to `_awaiting_capabilities`; creates ClientChatInfo entry; notifies all pregame chat clients of new joiner.
- **Calls**: `channel->enqueueOutgoingMessage()`, screen_printf callbacks.
- **Notes**: Version mismatch triggers disconnect; name is stored in Pascal string, converted to C string for chat info.

### Client::handleCapabilitiesMessage
- **Signature**: `void Client::handleCapabilitiesMessage(CapabilitiesMessage* capabilitiesMessage, CommunicationsChannel*)`
- **Purpose**: Gatherer receives joiner capabilities; check gatherable and send all known clients' info.
- **Inputs**: CapabilitiesMessage with capability flags and versions.
- **Outputs/Return**: None (sends ClientInfoMessage for all connected clients, transitions state).
- **Side effects**: Sets client state to `_connected_but_not_yet_shown` or `_ungatherable` based on capabilities_indicate_player_is_gatherable().
- **Calls**: `capabilities_indicate_player_is_gatherable()`.
- **Notes**: Called only in state `_awaiting_capabilities`; validates protocol, script, and version compatibility.

### Client::handleAcceptJoinMessage
- **Signature**: `void Client::handleAcceptJoinMessage(AcceptJoinMessage* acceptJoinMessage, CommunicationsChannel*)`
- **Purpose**: Joiner confirmed joining; add to topology and broadcast topology update to all players.
- **Inputs**: AcceptJoinMessage containing new player data.
- **Outputs/Return**: None (modifies topology, sends TopologyMessage).
- **Side effects**: Increments topology->player_count; sets client state to `_awaiting_map`; calls check_player callback; calls gatherCallbacks->JoinSucceeded().
- **Calls**: `NetUpdateTopology()`, `NetDistributeTopology(tagNEW_PLAYER)`.
- **Notes**: Called only in state `_awaiting_accept_join`; fills topology->players[player_count]; stream_id and dspAddress set from channel.

### Client::drop
- **Signature**: `void Client::drop()`
- **Purpose**: Handle client disconnection; remove from topology, clean up chat info, notify callbacks and other clients.
- **Inputs**: None (operates on this client's channel and state).
- **Outputs/Return**: None (broadcasts removal via ClientInfoMessage and TopologyMessage).
- **Side effects**: Erases client from topology if in `_awaiting_map`; broadcasts ClientInfoMessage with kRemove; calls gatherCallbacks->JoinedPlayerDropped() or JoiningPlayerDropped().
- **Calls**: `NetUpdateTopology()`, `NetDistributeTopology(tagDROPPED_PLAYER)`.
- **Notes**: Handles both pregame and in-game drops; asserts if in `_awaiting_map` but not found in topology.

### NetProcessMessagesInGame
- **Signature**: `void NetProcessMessagesInGame()`
- **Purpose**: During gameplay, pump all active network channels and dispatch incoming messages.
- **Inputs**: None.
- **Outputs/Return**: None.
- **Side effects**: Calls pump() and dispatchIncomingMessages() on all channels (joiner's server channel or all gatherer's client channels); also pumps MetaserverClient if connected.
- **Calls**: `CommunicationsChannel::pump()`, `CommunicationsChannel::dispatchIncomingMessages()`.
- **Notes**: Called each game frame; role-aware (gatherer vs joiner).

### NetChangeMap
- **Signature**: `bool NetChangeMap(struct entry_point *entry)`
- **Purpose**: On level change, gatherer retrieves and distributes new map; joiners wait for it.
- **Inputs**: Entry point (level info).
- **Outputs/Return**: true if map transfer successful, false otherwise.
- **Side effects**: Calls NetDistributeGameDataToAllPlayers() or NetReceiveGameData() depending on role; processes loaded map via process_net_map_data().
- **Calls**: `get_map_for_net_transfer()`, `NetDistributeGameDataToAllPlayers()`, `NetReceiveGameData()`, `process_net_map_data()`.
- **Notes**: Only gathers map if localPlayerIndex == sServerPlayerIndex; logError if server died; process_net_map_data() frees the WAD buffer.

### NetGatherPlayer
- **Signature**: `int NetGatherPlayer(const prospective_joiner_info &player, CheckPlayerProcPtr check_player)`
- **Purpose**: Gatherer sends JoinPlayerMessage to a joiner, assigning next identifier and setting state to await acceptance.
- **Inputs**: Joiner info (stream_id, name) and check_player callback.
- **Outputs/Return**: kGatherPlayerSuccessful.
- **Side effects**: Sends JoinPlayerMessage with next topology->nextIdentifier; transitions client to state `_awaiting_accept_join`; stores callback in Client::check_player.
- **Calls**: `channel->enqueueOutgoingMessage()`.
- **Notes**: Called from gather dialog loop; awaits AcceptJoinMessage response.

### Client::capabilities_indicate_player_is_gatherable
- **Signature**: `bool Client::capabilities_indicate_player_is_gatherable(bool warn_joiner)`
- **Purpose**: Validate joiner's capabilities against gatherer's requirements; send error message if incompatible.
- **Inputs**: warn_joiner ΓÇô whether to send ServerWarningMessage to joiner on failure.
- **Outputs/Return**: true if gatherable, false if protocol/Lua version mismatch or Gatherable flag is 0.
- **Side effects**: Sends ServerWarningMessage if warn_joiner && incompatible.
- **Calls**: `channel->enqueueOutgoingMessage()`, `getcstr()`.
- **Notes**: Checks Gatherable flag, Ring/Star support, and Lua/Netscript versions; protocol choice from network_preferences->game_protocol.

---

## Control Flow Notes

**Gatherer (server) flow:**
1. Gathers players in netGathering state via NetCheckForNewJoiner() loop (calls message handlers)
2. For each ready joiner, calls NetGatherPlayer() to send join request
3. Receives AcceptJoinMessage ΓåÆ adds to topology, broadcasts TopologyMessage
4. Before game start, calls NetDistributeGameDataToAllPlayers() to send map, physics, Lua
5. During gameplay, calls NetProcessMessagesInGame() each frame to handle messages, chat, topology updates

**Joiner flow:**
1. Begins in netConnecting; calls NetUpdateJoinState() repeatedly to attempt TCP connection with retry
2. On connection, enters netJoining; awaits HelloMessage and sends JoinerInfoMessage
3. Gathers capabilities from server ΓåÆ awaits AcceptJoinMessage
4. On acceptance, enters netWaiting; awaits map via NetReceiveGameData()
5. During gameplay, pumps server connection for messages and topology updates

**Ring vs. Star protocol:**
- netState does not distinguish; sCurrentGameProtocol is set at game start to either sRingGameProtocol or sStarGameProtocol
- Each protocol implements PacketHandler(), GetNetTime(), PeekUnconfirmedActionFlag(), UpdateUnconfirmedActionFlags(), GetUnconfirmedActionFlagsCount()
- NetGetLatency() dispatches to active protocol's latency functions

**Message handling:**
- MessageDispatcher on each Client routes incoming message types to registered handler methods
- Chat, color changes, and topology updates propagate via ClientInfoMessage and TopologyMessage
- Map/physics/Lua streamed via PhysicsMessage, MapMessage, LuaMessage (and zipped variants)

## External Dependencies

- **SDL_net**: TCP/UDP socket abstraction (TCPsocket, UDPsocket, SDLNet_SocketSet)
- **Message classes**: JoinerInfoMessage, CapabilitiesMessage, AcceptJoinMessage, TopologyMessage, etc. (defined in network_messages.h)
- **CommunicationsChannel**: TCP message framing and buffering (defined elsewhere)
- **NetworkGameProtocol, RingGameProtocol, StarGameProtocol**: Protocol state machines
- **ConsoleCallbacks, GatherCallbacks, ChatCallbacks**: Application-level feedback hooks
- **MetaserverClient**: Metaserver registration and updates
- **Logging**: logAnomaly(), logError(), logNote() for anomaly/debug reporting
- **Progress UI**: open_progress_dialog(), set_progress_dialog_message(), draw_progress_bar(), close_progress_dialog()
- **Preferences**: network_preferences, player_preferences, environment_preferences (global)
- **map.h**: entry_point, TICKS_PER_SECOND, get_map_for_net_transfer(), process_net_map_data()
- **game_errors.h, game_data/game_info**: Game state structures (opaque void* in this file)
- **Lua**: LoadLuaScript() for netscript loading
- **ConnectPool**: Nonblocking TCP connection pooling with retry/resolution

**Defined elsewhere but called here:**
- NetSync(), NetUnSync() ΓÇô synchronization primitives
- NetGetLocalPlayerIndex(), NetGetPlayerIdentifier(), NetGetNumberOfPlayers() ΓÇô query topology
- NetGetPlayerData(), NetGetGameData() ΓÇô query game state
- process_action_flags() ΓÇô dispatch action flags from packets to game logic
- screen_printf(), alert_user() ΓÇô UI feedback
- machine_tick_count() ΓÇô timing
