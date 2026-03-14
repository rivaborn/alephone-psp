# Source_Files/Network/network_star_spoke.cpp

## File Purpose
Implements the spoke (client) side of the star topology network protocol for Aleph One multiplayer games. Handles game data packet exchange with the hub, action flags queuing/acknowledgement, lossy byte stream distribution, and dynamic timing synchronization.

## Core Responsibilities
- Initialize/cleanup spoke networking state and timer task
- Process incoming game data packets from hub (CRC validation, message dispatch, action flags deserialization)
- Queue and transmit local player action flags to hub
- Track and dequeue hub acknowledgements for sent flags
- Manage per-player action flag queues and net-dead state
- Measure and apply timing adjustments for clock synchronization
- Buffer and distribute lossy byte streams to specified player sets
- Maintain rolling-window latency statistics (30 ticks / ~1 second)
- Handle NAT-friendly identification packets (spokes identify themselves to hub)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpokePreferences` | struct | Configuration: net-death timeouts, recovery period, timing window size/nth-element |
| `NetworkPlayer_spoke` | struct | Per-player state: zombie flag, connected status, net-dead tick, action queue pointer |
| `IncomingGameDataPacketProcessingContext` | struct | Transient packet processing state: whether messages are done, whether timing adjustment received |
| `SpokeLossyByteStreamChunkDescriptor` | struct | Outgoing lossy stream metadata: length, message type, destination bitmask |
| `StarMessageHandler` | typedef | Function pointer: `void (*)(AIStream&, IncomingGameDataPacketProcessingContext&)` |
| `MessageTypeToMessageHandler` | typedef | `std::map<uint16, StarMessageHandler>` for dispatching message types |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSpokePreferences` | `SpokePreferences` | static | Configuration parameters (can be overridden via XML) |
| `sOutgoingFlags` | `TickBasedActionQueue` | static | Flags sent to hub, awaiting ACK |
| `sUnconfirmedFlags` | `TickBasedActionQueue` | static | Local player flags not yet sent/confirmed |
| `sLocallyGeneratedFlags` | `DuplicatingTickBasedCircularQueue<action_flags_t>` | static | Composite queue feeding outgoing and unconfirmed |
| `sNetworkPlayers` | `vector<NetworkPlayer_spoke>` | static | Per-player state (all players including self) |
| `sNetworkTicker` | `int32` | static | Monotonic tick counter incremented each timer callback |
| `sLastNetworkTickHeard` | `int32` | static | Last network tick when packet received from hub |
| `sLastNetworkTickSent` | `int32` | static | Last network tick when packet sent to hub |
| `sConnected` | `bool` | static | Hub connection status |
| `sSpokeActive` | `bool` | static | Whether spoke is currently active (gates packet processing) |
| `sSpokeTickTask` | `myTMTaskPtr` | static | Timer task handle |
| `sHubAddress` | `NetAddrBlock` | static | Hub network address |
| `sLocalPlayerIndex` | `size_t` | static | This spoke's player index in the game |
| `sSmallestUnreceivedTick` | `int32` | static | Smallest tick for which we haven't received action flags from hub |
| `sSmallestUnconfirmedTick` | `int32` | static | Smallest tick in unconfirmed queue not yet enqueued to player queue |
| `sNthElementFinder` | `WindowedNthElementFinder<int32>` | static | Windowed latency measurements (finds median-like value) |
| `sTimingMeasurementValid` | `bool` | static | Whether latency window is full enough to use measurement |
| `sTimingMeasurement` | `int32` | static | Current latency measurement in ticks |
| `sOutgoingLossyByteStreamData` | `CircularByteBuffer` | static | Raw bytes for outgoing lossy streams |
| `sOutgoingLossyByteStreamDescriptors` | `CircularQueue<SpokeLossyByteStreamChunkDescriptor>` | static | Metadata for each chunk in buffer |
| `sDisplayLatencyBuffer` | `vector<int32>` | static | Rolling 30-tick latency history |
| `sDisplayLatencyCount` | `uint32` | static | Total samples enqueued (for averaging) |
| `sDisplayLatencyTicks` | `int32` | static | Sum of recent latencies |
| `sMessageTypeToMessageHandler` | `MessageTypeToMessageHandler` | static | Dispatch map for message types |
| `sRequestedTimingAdjustment` | `int8` | static | Requested timing adjustment from hub |
| `sOutstandingTimingAdjustment` | `int8` | static | Remaining timing adjustment to apply this frame |

## Key Functions / Methods

### spoke_initialize
- **Signature:** `void spoke_initialize(const NetAddrBlock& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnected[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- **Purpose:** Initialize all spoke state, queues, timers, and preferences.
- **Inputs:** Hub address, starting tick, player count, per-player action queues, connected flags, local player index, whether hub is local (loopback).
- **Outputs/Return:** None (void).
- **Side effects:** Creates and starts timer task (`sSpokeTickTask`), allocates network player state, initializes all queues, clears message handlers and populates with defaults, sets `sConnected = true`.
- **Calls:** `NetDDPNewFrame()`, `myXTMSetup()`, message handler registration.
- **Notes:** Asserts that local player is connected and has a non-null queue. Resets display latency buffer and measurement tracking.

### spoke_cleanup
- **Signature:** `void spoke_cleanup(bool inGraceful)`
- **Purpose:** Gracefully shut down spoke networking.
- **Inputs:** Whether cleanup is graceful (currently unused in observed code).
- **Outputs/Return:** None (void).
- **Side effects:** Stops timer task (under mutex), sets `sSpokeActive = false`, sends final packet, clears all queues and player state, releases frame.
- **Calls:** `take_mytm_mutex()`, `myTMRemove()`, `send_packet()`, `check_send_packet_to_hub()`, `release_mytm_mutex()`, `myTMCleanup()`, `NetDDPDisposeFrame()`.
- **Notes:** Takes mutex to ensure no packet is mid-processing when timer stops.

### spoke_get_net_time
- **Signature:** `int32 spoke_get_net_time()`
- **Purpose:** Return the current network time, adjusted for measured latency if connected.
- **Inputs:** None.
- **Outputs/Return:** Network time (ticks); if connected and timing valid, returns write tick of outgoing queue minus latency measurement; otherwise returns local player queue write tick.
- **Side effects:** Logs on first call or when delay changes.
- **Calls:** Logging functions.
- **Notes:** Used by game logic to synchronize to network time. Static `sPreviousDelay` avoids spam logging.

### spoke_distribute_lossy_streaming_bytes_to_everyone
- **Signature:** `void spoke_distribute_lossy_streaming_bytes_to_everyone(int16 inDistributionType, byte* inBytes, uint16 inLength, bool inExcludeLocalPlayer, bool onlySendToTeam)`
- **Purpose:** Queue outgoing lossy stream bytes for broadcast, optionally filtered by team.
- **Inputs:** Distribution type, byte buffer, length, exclude-local flag, team-only flag.
- **Outputs/Return:** None (void).
- **Side effects:** Computes destination bitmask based on player connectivity and team (if applicable), then calls `spoke_distribute_lossy_streaming_bytes()`.
- **Calls:** `NetGetPlayerData()`, `spoke_distribute_lossy_streaming_bytes()`.
- **Notes:** Skips zombie players and disconnected players. If `onlySendToTeam`, matches local player's team.

### spoke_distribute_lossy_streaming_bytes
- **Signature:** `void spoke_distribute_lossy_streaming_bytes(int16 inDistributionType, uint32 inDestinationsBitmask, byte* inBytes, uint16 inLength)`
- **Purpose:** Queue outgoing lossy stream bytes with explicit destination bitmask.
- **Inputs:** Distribution type, destination bitmask (bit per player), byte buffer, length.
- **Outputs/Return:** None (void).
- **Side effects:** Enqueues bytes to circular buffer and descriptor to descriptor queue. Logs warnings if buffers are full (data discarded).
- **Calls:** Logging functions, buffer enqueue methods.
- **Notes:** Does **not** fail gracefully if buffers are fullΓÇödata is silently discarded with a log warning. Called by frame/spoke tick logic.

### spoke_received_network_packet
- **Signature:** `void spoke_received_network_packet(DDPPacketBufferPtr inPacket)`
- **Purpose:** Entry point for incoming DDP packets; validates CRC and dispatches to packet handler.
- **Inputs:** Received DDP packet buffer.
- **Outputs/Return:** None (void).
- **Side effects:** Parses magic, CRC; if valid, dispatches to `spoke_received_game_data_packet_v1()` based on magic (V1 or V1-with-reflected-flags).
- **Calls:** `spoke_received_game_data_packet_v1()`, logging and CRC functions.
- **Notes:** Returns early if not connected or spoke inactive. Ignores unknown packet types and suppresses exceptions from malformed packets.

### spoke_received_game_data_packet_v1
- **Signature:** `static void spoke_received_game_data_packet_v1(AIStream& ps, bool reflected_flags)`
- **Purpose:** Process incoming game data packet: ACK flags, dispatch messages, deserialize and enqueue action flags, update latency.
- **Inputs:** Deserialization stream, whether this packet reflects the spoke's own flags back.
- **Outputs/Return:** None (void).
- **Side effects:** Sets `sHeardFromHub = true`; dequeues acknowledged flags from `sOutgoingFlags`; processes embedded messages; enqueues received flags to player queues; updates latency window; handles special "we are alone" case (only connected player).
- **Calls:** `process_messages()`, `sNthElementFinder.insert()`, logging functions, queue enqueue/dequeue.
- **Notes:** Validates ACK is not ahead of outgoing write tick (or allows if reflected flags). Complex logic to handle net-dead players and partially-filled packets.

### spoke_tick
- **Signature:** `static bool spoke_tick()`
- **Purpose:** Timer callback (~30 Hz); handles timing adjustments, queues local flags, sends packets.
- **Inputs:** None (called by timer).
- **Outputs/Return:** `true` (continue calling).
- **Side effects:** Increments `sNetworkTicker`; checks connection timeout and marks disconnected if silent too long; processes timing adjustments (positive = skip ticks, negative = enqueue extra); queues local input flags via `parse_keymap()`; sends packets (if new data or recovery period); enqueues net-dead flags for other players if disconnected.
- **Calls:** `parse_keymap()`, `send_packet()`, `send_identification_packet()`, `check_send_packet_to_hub()`, logging.
- **Notes:** Before connection heard from hub, sends identification packets every ~30 ticks (1 sec). Complex logic around which queue to enqueue to (outgoing vs. locally-generated) based on game phase (pregame vs. in-game).

### send_packet
- **Signature:** `static void send_packet()`
- **Purpose:** Serialize and transmit game data packet to hub.
- **Inputs:** None (uses static state).
- **Outputs/Return:** None (void).
- **Side effects:** Builds packet with header (magic, CRC), ACK (smallest unread tick), any pending lossy stream message, end-of-messages marker, outgoing flags; computes CRC; updates `sLastNetworkTickSent`; sends via DDP or local loopback.
- **Calls:** `AOStreamBE` operators, `calculate_data_crc_ccitt()`, `NetDDPSendFrame()`, `send_frame_to_local_hub()`.
- **Notes:** Lossy stream chunk is dequeued before writing to packet to avoid partial data on buffer exhaustion. Suppresses exceptions.

### send_identification_packet
- **Signature:** `static void send_identification_packet()`
- **Purpose:** Send NAT identification packet with local player index.
- **Inputs:** None.
- **Outputs/Return:** None (void).
- **Side effects:** Builds and sends small packet with magic `kSpokeToHubIdentification` and local player index.
- **Calls:** `AOStreamBE` operators, `calculate_data_crc_ccitt()`, `NetDDPSendFrame()`, `send_frame_to_local_hub()`.
- **Notes:** Used at startup (before first game packet) so hub can associate packet source address with player ID. Suppresses exceptions.

### spoke_latency
- **Signature:** `int32 spoke_latency()`
- **Purpose:** Return rolling-average latency in milliseconds.
- **Inputs:** None.
- **Outputs/Return:** Latency in ms (1/1000 s), or `kNetLatencyInvalid` if buffer has fewer than 30 samples.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Computes `sDisplayLatencyTicks * 1000 / TICKS_PER_SECOND / buffer_size` if at least 30 samples accumulated.

### DefaultSpokePreferences / WriteSpokePreferences
- **Signature:** `void DefaultSpokePreferences()` / `void WriteSpokePreferences(FILE* F)`
- **Purpose:** Reset preferences to hardcoded defaults / serialize preferences to XML.
- **Inputs:** File pointer (for write).
- **Outputs/Return:** None (void).
- **Side effects:** Modifies `sSpokePreferences` or writes fprintf output.
- **Calls:** fprintf (for write).

### Spoke_GetParser
- **Signature:** `XML_ElementParser* Spoke_GetParser()`
- **Purpose:** Return XML configuration parser for spoke preferences.
- **Inputs:** None.
- **Outputs/Return:** Pointer to global `SpokeConfigurationParser` instance.
- **Side effects:** None.

## Control Flow Notes

**Initialization ΓåÆ Running ΓåÆ Cleanup:**
1. Game calls `spoke_initialize()` ΓåÆ sets up state, starts timer.
2. Timer fires every ~33 ms, calling `spoke_tick()`:
   - Dequeues local input, handles timing adjustments.
   - Sends packets when needed.
3. Network packets arrive asynchronously ΓåÆ `spoke_received_network_packet()` ΓåÆ `spoke_received_game_data_packet_v1()`:
   - Dequeues ACKs, enqueues received flags.
4. Game calls `spoke_cleanup()` ΓåÆ stops timer, sends final packet, frees resources.

**Timing Adjustment Loop:**
- Hub measures round-trip latency, sends timing adjustment message.
- Spoke applies adjustment by skipping ticks (positive) or enqueueing extra (negative) in `spoke_tick()`.
- Latency window measures successive round-trip times; median-like value smooths spikes.

**"We Are Alone" Case:**
- If spoke is only connected player and receives an empty-flags packet from hub, it enqueues net-dead flags for other players without waiting for them.

## External Dependencies
- **Network layer:** `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()` (defined elsewhere).
- **Timers:** `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take/release_mytm_mutex()` from `mytm.h`.
- **Serialization:** `AIStreamBE`, `AOStreamBE` from `AStream.h`.
- **Utilities:** `WindowedNthElementFinder<int32>`, `CircularByteBuffer`, `CircularQueue<T>` from headers.
- **Game logic callbacks:** `make_player_really_net_dead(size_t)`, `call_distribution_response_function_if_available(byte*, uint16, int16, uint8)` (declared extern).
- **Input:** `parse_keymap()` from `vbl.h`.
- **Player data:** `NetGetPlayerData(size_t)` returns `player_info*` for team/player queries.
- **Logging:** `logContextNMT()`, `logDumpNMT*()`, `logWarningNMT*()`, etc.
- **CRC:** `calculate_data_crc_ccitt(byte*, uint32)`.
- **Protocol constants:** `kPROTOCOL_TYPE` from `network_private.h`; message types from `network_star.h`.
