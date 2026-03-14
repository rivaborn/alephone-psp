# Source_Files/Network/network_star_hub.cpp

## File Purpose
Implements the hub component of a star-topology network protocol for Aleph One game engine. Manages packet routing, player state, action flag distribution, and network timing synchronization across connected game clients (spokes).

## Core Responsibilities
- Hub initialization, cleanup, and periodic tick processing
- Receiving and parsing game data packets from remote players (spokes)
- Tracking player connectivity, acknowledgments, and netdead status
- Distributing action flags to all connected players with timing adjustments
- Managing lossy byte stream data buffering and routing
- Detecting player timeout and enforcing network death
- Bandwidth reduction through recovery sends and late flag handling

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `HubPreferences` | struct | Configuration for window sizes, nth-element selection, death timeouts, send periods, bandwidth reduction |
| `NetworkPlayer_hub` | struct | Per-player state: address, connectivity, last heard tick, ACK tracking, nth-element finder, timing adjustment, netdead tick, latency buffer |
| `HubLossyByteStreamChunkDescriptor` | struct | Metadata for lossy data chunks: length, type, destination bitmask, sender index |
| `XML_HubConfigurationParser` | class | XML parser for hub configuration attributes |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sHubPreferences` | `HubPreferences` | static | Hub behavior configuration (windows, timeouts, send periods) |
| `sNetworkTicker` | `int32` | static | Monotonic tick counter, advances regardless of game clock |
| `sLastNetworkTickSent` | `int32` | static | Last tick at which packets were sent (for recovery sends) |
| `sSmallestRealGameTick` | `int32` | static | First in-game tick (pre-game ticks are earlier) |
| `sSmallestPostGameTick` | `int32` | static | First tick after game ends; signals shutdown completion |
| `sFlagsQueues` | `TickBasedActionQueueCollection` | static | Per-player circular queues of action flags |
| `sPlayerDataDisposition` | `MutableElementsTickBasedCircularQueue<uint32>` | static | Bitmask tracking which players have acknowledged each tick |
| `sConnectedPlayersBitmask` | `uint32` | static | Bitmask of currently connected players |
| `sLaggingPlayersBitmask` | `uint32` | static | Bitmask of players waiting on data; triggers synthetic flag generation |
| `sNetworkPlayers` | `NetworkPlayerCollection` | static | Per-player state array |
| `sAddressToPlayerIndex` | `AddressToPlayerIndexType` | static | Map from network address to player index (for NAT-friendly ID lookup) |
| `sLocalPlayerIndex` | `size_t` | static | Index of local spoke on hub machine (or `NONE` for standalone) |
| `sOutgoingLossyByteStreamData` | `CircularByteBuffer` | static | Buffer for outgoing lossy byte stream chunks |
| `sOutgoingLossyByteStreamDescriptors` | `CircularQueue<...>` | static | Metadata queue for lossy data chunks |

## Key Functions / Methods

### hub_initialize
- **Signature:** `void hub_initialize(int32 inStartingTick, size_t inNumPlayers, const NetAddrBlock* const* inPlayerAddresses, size_t inLocalPlayerIndex)`
- **Purpose:** Initialize hub state, create tick task, reset all queues and player structures.
- **Inputs:** Starting game tick, number of players, optional player addresses (may be NULL), local player index.
- **Outputs/Return:** None.
- **Side effects:** Allocates DDPFrame, resets all static queues, starts myTMTask for periodic ticking.
- **Calls:** `NetDDPNewFrame()`, `myXTMSetup()`, queue reset methods.
- **Notes:** Sets `sHubActive = true` and `sHubInitialized = true` upon success. Does not require addresses in NAT-friendly mode (players send identification packets instead).

### hub_cleanup
- **Signature:** `void hub_cleanup(bool inGraceful, int32 inSmallestPostGameTick)`
- **Purpose:** Gracefully or forcefully shut down hub, wait for acknowledgments, deallocate resources.
- **Inputs:** Graceful flag (wait for ACKs vs. immediate stop), post-game tick threshold.
- **Outputs/Return:** None.
- **Side effects:** Sets `sSmallestPostGameTick`, calls `hub_check_for_completion()`, sets `sHubActive = false`, removes tick task, clears queues and frame.
- **Calls:** `take_mytm_mutex()`, `hub_check_for_completion()`, `myTMRemove()`, `myTMCleanup()`, `NetDDPDisposeFrame()`.
- **Notes:** In graceful mode, spins waiting for `sHubActive` to become false (set by packet handler when all ACKs received). Non-graceful immediately stops processing.

### hub_received_network_packet
- **Signature:** `void hub_received_network_packet(DDPPacketBufferPtr inPacket)`
- **Purpose:** Entry point for incoming network packets; routes to appropriate handler based on packet magic number.
- **Inputs:** Packet buffer with datagram.
- **Outputs/Return:** None.
- **Side effects:** Parses packet header, validates CRC, dispatches to `hub_received_game_data_packet_v1()` or `hub_received_identification_packet()`. Calls `check_send_packet_to_spoke()` at end.
- **Calls:** `AIStreamBE` constructor, CRC validation, packet-type handlers.
- **Notes:** Returns early if `!sHubActive`. CRC check clears bytes 2ΓÇô3 in packet before validation. Exception-safe (try-catch discards malformed packets).

### hub_received_game_data_packet_v1
- **Signature:** `static void hub_received_game_data_packet_v1(AIStream& ps, int inSenderIndex)`
- **Purpose:** Process incoming action flags and messages from a spoke; update ACKs, enqueue flags, handle late-arriving flags.
- **Inputs:** Input stream positioned after packet header, sender player index.
- **Outputs/Return:** None.
- **Side effects:** Calls `player_acknowledged_up_to_tick()`, updates `sFlagsQueues` and `sLateFlagsQueues`, updates `sLastFlagsReceived`, marks players in `sPlayerReflectedFlags` if needed. May call `player_provided_flags_from_tick_to_tick()` to trigger sends.
- **Calls:** `player_acknowledged_up_to_tick()`, `process_messages()`, `player_provided_flags_from_tick_to_tick()`.
- **Notes:** Validates ACK is not too far ahead; skips redundant flags; enqueues late flags into separate queue; tracks reflected flags for spokes that need echo-back.

### hub_tick
- **Signature:** `static bool hub_tick()`
- **Purpose:** Periodic update called by mytm task; check for netdead players, trigger recovery sends if needed, send packets, invoke local spoke.
- **Inputs:** None (uses static state).
- **Outputs/Return:** Always `true` (continue ticking).
- **Side effects:** Increments `sNetworkTicker`. Calls `make_player_netdead()` for timeout players, `send_packets()` if needed, `check_send_packet_to_spoke()` for local spoke. Updates `sLastNetworkTickSent`.
- **Calls:** `make_player_netdead()`, `send_packets()`, `check_send_packet_to_spoke()`, `make_up_flags_for_first_incomplete_tick()`.
- **Notes:** Handles both incremental and recovery (bandwidth-reduced) send strategies based on latency and minimum send period. Not reentrant; protected by mytm mutex.

### send_packets
- **Signature:** `static void send_packets()`
- **Purpose:** Construct and transmit game data packets to all connected players; encode ACKs, messages, action flags.
- **Inputs:** None (uses static state and queues).
- **Outputs/Return:** None.
- **Side effects:** Increments `sFlagSendTimeQueue`, dequeues lossy byte stream data, updates per-player `mLastRecoverySend` and latency buffers, calls `NetDDPSendFrame()` or `send_frame_to_local_spoke()`, dequeues sent lossy descriptors.
- **Calls:** `AOStreamBE`, CRC calculation, `NetDDPSendFrame()`, `send_frame_to_local_spoke()`.
- **Notes:** Bandwidth reduction selects recovery vs. incremental based on effective latency. Constructs tick-major flag array. Reflects flags for players that need them. Handles netdead ticks (stops sending after death).

### player_provided_flags_from_tick_to_tick
- **Signature:** `static bool player_provided_flags_from_tick_to_tick(size_t inPlayerIndex, int32 inFirstNewTick, int32 inSmallestUnreceivedTick)`
- **Purpose:** Mark ticks as received from player; advance completion queue when all players have provided data for a tick.
- **Inputs:** Player index, range of new ticks.
- **Outputs/Return:** `true` if any new tick became complete.
- **Side effects:** Updates `sPlayerDataDisposition` bitmask, advances `sSmallestIncompleteTick`, clears late queue for player, updates `sLastRealUpdate`.
- **Calls:** (none visible in signature; operates on static queues).
- **Notes:** Masks out player bit in disposition; when bitmask reaches 0, tick is complete and ready to send. Dequeues late flags for this player.

### make_player_netdead
- **Signature:** `static void make_player_netdead(int inPlayerIndex)`
- **Purpose:** Mark a player as disconnected and pretend they've provided/acknowledged all remaining data to unblock the hub.
- **Inputs:** Player index.
- **Outputs/Return:** None.
- **Side effects:** Sets `mNetDeadTick`, clears `mConnected`, updates bitmasks, calls `player_provided_flags_from_tick_to_tick()` and `player_acknowledged_up_to_tick()`.
- **Calls:** `player_provided_flags_from_tick_to_tick()`, `player_acknowledged_up_to_tick()`.
- **Notes:** Effectively removes player from contention so remaining players can continue.

### hub_latency
- **Signature:** `int32 hub_latency(int player_index)`
- **Purpose:** Query measured network latency to a remote player in milliseconds.
- **Inputs:** Player index.
- **Outputs/Return:** Latency in ms, or `kNetLatencyInvalid` / `kNetLatencyDisconnected`.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Reads from `mDisplayLatencyBuffer` and `mDisplayLatencyTicks` (populated during packet send). Returns invalid for local player.

## Control Flow Notes

**Initialization phase:** `hub_initialize()` sets up queues and starts the tick task. Players send identification packets (NAT-friendly mode) mapping their address to their index.

**Game phase:** Periodic `hub_tick()` (every ~33ms at 30 TICKS_PER_SECOND) checks for netdead players and decides whether to send. Incoming `hub_received_network_packet()` calls (from network thread) parse ACKs and flags, updating queues. When all players ACK a tick, the hub advances `sSmallestIncompleteTick` and prepares to send the next batch.

**Shutdown phase:** `hub_cleanup(true)` sets `sSmallestPostGameTick` and waits for all remaining ACKs via `hub_check_for_completion()`. Once satisfied, sets `sHubActive = false`.

## External Dependencies

- **Includes:** `network_star.h`, `TickBasedCircularQueue.h`, `network_private.h`, `mytm.h`, `AStream.h`, `Logging.h`, `WindowedNthElementFinder.h`, `CircularByteBuffer.h`, `XML_ElementParser.h`, `SDL_timer.h`, `crc.h`, `player.h`
- **External symbols:** `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()`, `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `spoke_received_network_packet()`, `calculate_data_crc_ccitt()`, `local_random()`, logging macros
