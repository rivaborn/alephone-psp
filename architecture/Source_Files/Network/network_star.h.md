# Source_Files/Network/network_star.h

## File Purpose
Defines the hub-and-spoke network architecture interface for Marathon: Aleph One multiplayer games. Manages both hub (server/coordinator) and spoke (client) networking roles, including packet handling, tick synchronization, action flag distribution, and lossy streaming data channels.

## Core Responsibilities
- Define message type constants for star topology protocol communication
- Declare hub-side functions: initialization, packet receiving, graceful shutdown, preferences management
- Declare spoke-side functions: initialization, packet receiving, timing queries, lossy streaming, shutdown
- Manage tick-based action flag queues for synchronized player input distribution
- Measure and report network latency for both hub-to-spoke and spoke-to-spoke (via hub) connections
- Support lossy streaming channels (e.g., voice/microphone data) with selective recipient targeting
- Provide XML-based preference serialization for both hub and spoke configurations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| action_flags_t | typedef | 32-bit player input flags value type |
| TickBasedActionQueue | typedef | Read-only tick-indexed circular queue of action flags |
| WritableTickBasedActionQueue | typedef | Write-only interface to tick-indexed action flag queues |

## Global / File-Static State
None.

## Key Functions / Methods

### hub_initialize
- Signature: `void hub_initialize(int32 inStartingTick, size_t inNumPlayers, const NetAddrBlock* const* inPlayerAddresses, size_t inLocalPlayerIndex)`
- Purpose: Initialize hub (server/coordinator) for star network game
- Inputs: Starting game tick, number of players, array of player network addresses, local player index
- Outputs/Return: None
- Side effects: Allocates hub state, begins accepting packets from spokes
- Calls: (Not visible in header)
- Notes: Must be called before any spoke packets are received

### spoke_initialize
- Signature: `void spoke_initialize(const NetAddrBlock& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnectedStatus[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- Purpose: Initialize spoke (client) for star network game
- Inputs: Hub address, first game tick, player count, writable action queue array (to be filled), connection status array, local player index, hub-is-local flag
- Outputs/Return: None
- Side effects: Allocates spoke state, establishes initial contact with hub
- Calls: (Not visible in header)
- Notes: Action queues will receive server-distributed action flags; inHubIsLocal may optimize LAN play

### hub_received_network_packet, spoke_received_network_packet
- Signature: `void hub_received_network_packet(DDPPacketBufferPtr inPacket); void spoke_received_network_packet(DDPPacketBufferPtr inPacket)`
- Purpose: Process incoming UDP packets at hub or spoke
- Inputs: Network packet buffer
- Outputs/Return: None
- Side effects: Updates game state, queues, timing, or relays packets to other spokes (hub only)
- Calls: (Not visible in header)

### spoke_distribute_lossy_streaming_bytes_to_everyone
- Signature: `void spoke_distribute_lossy_streaming_bytes_to_everyone(int16 inDistributionType, byte* inBytes, uint16 inLength, bool inExcludeLocalPlayer, bool onlySendToTeam)`
- Purpose: Broadcast lossy real-time streaming data (voice, microphone) to all or team players
- Inputs: Distribution type code, byte buffer, length, exclude-self flag, team-only flag
- Outputs/Return: None
- Side effects: Sends UDP packet to hub for distribution
- Calls: (Not visible in header)

### spoke_latency, hub_latency
- Signature: `int32 spoke_latency(); int32 hub_latency(int player_index)`
- Purpose: Query measured network latency (round-trip milliseconds)
- Inputs: (hub_latency) player index
- Outputs/Return: Latency in ms; kNetLatencyInvalid if unmeasured; kNetLatencyDisconnected if disconnected (hub only)
- Side effects: None
- Calls: (Not visible in header)

### spoke_get_net_time, spoke_get_unconfirmed_flags_queue, spoke_get_smallest_unconfirmed_tick
- Signature: `int32 spoke_get_net_time(); TickBasedActionQueue* spoke_get_unconfirmed_flags_queue(); int32 spoke_get_smallest_unconfirmed_tick()`
- Purpose: Query network timing state and unacknowledged local action flags for synchronization/prediction
- Inputs: None
- Outputs/Return: Game tick or queue pointer
- Side effects: None
- Calls: (Not visible in header)

**Other functions**: `hub_cleanup`, `spoke_cleanup` (graceful shutdown); `Hub_GetParser`, `Spoke_GetParser`, `DefaultHubPreferences`, `WriteSpokePreferences`, `DefaultSpokePreferences` (XML preferences).

## Control Flow Notes
Hub-spoke topology fits into the frame-based game loop:
- **Pregame**: `kPregameTicks` (90 ticks = 3 seconds at 30 fps) used for synchronization before real gameplay
- **Main loop**: Spokes enqueue local action flags; hub receives, accumulates, and broadcasts confirmed action flags to all spokes each tick
- **Timing**: `kTimingAdjustmentMessageType` messages sync tick counters across network
- **Lossy streaming**: Bypasses ordered action queues; sent in-band via `kSpokeToHubLossyByteStreamMessageType` / `kHubToSpokeLossyByteStreamMessageType` for low-latency voice/data
- **Connection tracking**: Hub monitors spoke connection status; latency measured bidirectionally

## External Dependencies
- **TickBasedCircularQueue.h**: Template classes `ConcreteTickBasedCircularQueue<T>`, `WritableTickBasedCircularQueue<T>` for tick-indexed action buffers
- **ActionQueues.h**: Related action queue management (referenced but not directly used in this header)
- **sdl_network.h**: Network primitives (`DDPPacketBufferPtr`, `NetAddrBlock`, UDP socket API)
- **map.h** (conditional): Defines `TICKS_PER_SECOND` (30); skipped in standalone hub builds
- **XML_ElementParser**: Preferences parser (declared; defined elsewhere)
- **Standard C**: stdio.h for file I/O in preference serialization
