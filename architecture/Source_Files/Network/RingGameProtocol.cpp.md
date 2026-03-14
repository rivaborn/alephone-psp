# Source_Files/Network/RingGameProtocol.cpp

## File Purpose
Implements the old ring-topology network game protocol for Aleph One (Marathon engine). Players arranged in a ring pass action-flag packets sequentially to a server; the server broadcasts timing and synchronization to coordinate gameplay across all clients in the network.

## Core Responsibilities
- Maintain ring topology state (upring/downring neighbors, sequence tracking)
- Queue and transmit local player action flags via ring packets with ACK/retry
- Synchronize game startup (ring must complete two cycles before game starts)
- Implement adaptive latency tuning to balance responsiveness vs. jitter tolerance
- Handle player dropout by removing dead players from the ring and reassigning server role if needed
- Process incoming ring packets and distribute action flags to player queues
- Manage three background task types: resend (ACK timeout), server, and queueing (local input capture)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NetStatus` | struct | Holds ring state: neighbor addresses, sequence counters, timing, flags-per-packet, server role, local net time |
| `NetQueue` | struct | Circular queue (size `NET_QUEUE_SIZE`) storing local action flags pending transmission |
| `RingPreferences` | struct | XML-configurable settings: accept packets from anyone, adapt-to-latency flag, latency hold period |
| `XML_RingConfigurationParser` | class | Parses `<ring_protocol>` XML elements; sets `sRingPreferences` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `status` | `NetStatusPtr` | static | Current ring network state (malloc'd at Enter, freed at Exit2) |
| `ringFrame`, `ackFrame`, `distributionFrame` | `DDPFramePtr` | static | Pre-allocated DDP frames for sending packets |
| `unpackedReceiveBuffer` | `char[ddpMaxData]` | static | Temporary buffer for unpacked incoming packets (format conversion) |
| `local_queue` | `volatile NetQueue` | static | Circular queue of local player action flags |
| `sRingPreferences` | `RingPreferences` | static | User/MML settings for protocol behavior |
| `resendTMTask`, `serverTMTask`, `queueingTMTask` | `myTMTaskPtr` | static | Background timer tasks for retransmit, server processing, local input capture |
| `sCurrentAdaptiveLatency` | `volatile int` | static | Current required action flags per packet (adapted dynamically) |
| `sAdaptiveLatencySamples[kAdaptiveLatencyWindowSize]` | `int[]` | static | Ring latency measurements for adaptive adjustment (NETWORK_ADAPTIVE_LATENCY_3) |
| `net_stats` | `network_statistics` | static | Debug counters (only if `DEBUG_NET` defined) |

## Key Functions / Methods

### RingGameProtocol::Enter
- **Signature:** `bool Enter(short* inNetStatePtr)`
- **Purpose:** Initialize protocol: allocate status structure and DDP frames, clear debug stats
- **Inputs:** `inNetStatePtr` ΓÇô pointer to network state variable (will be set to `netStartingUp` then `netActive`)
- **Outputs/Return:** `true` if all allocations succeeded; `false` on malloc failure
- **Side effects:** Allocates `status` and frame structures; opens debug stream if `STREAM_NET` enabled
- **Calls:** `malloc`, `NetDDPNewFrame` (3x), `obj_clear`
- **Notes:** Called before any other protocol operation; must be paired with `Exit1()` then `Exit2()`

### RingGameProtocol::Exit1
- **Signature:** `void Exit1()`
- **Purpose:** Stop background tasks before shutdown
- **Side effects:** Removes `resendTMTask`, `serverTMTask`, `queueingTMTask`
- **Calls:** `myTMRemove` (3x)
- **Notes:** Must be called before `Exit2()`; safe to call multiple times (tasks become NULL after first removal)

### RingGameProtocol::Exit2
- **Signature:** `void Exit2()`
- **Purpose:** Free all resources after tasks are stopped
- **Side effects:** Frees `status->buffer` and `status` pointer; disposes DDP frames; closes debug files
- **Calls:** `NetDDPDisposeFrame` (3x), `free`, `close_stream_file`

### RingGameProtocol::Sync
- **Signature:** `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- **Purpose:** Synchronize all players before game starts; server sends first ring packet, all players wait for ring to complete twice
- **Inputs:** 
  - `inTopology` ΓÇô player list and ring topology
  - `inSmallestGameTick` ΓÇô current world tick (for resuming saved games)
  - `inLocalPlayerIndex` ΓÇô this machine's player index
  - `inServerPlayerIndex` ΓÇô index of server player
- **Outputs/Return:** `true` if sync completed; `false` on timeout (50 seconds)
- **Side effects:** Initializes local net time, action flags per packet, queue indices, adaptive latency state; sets `*sNetStatePtr` to `netStartingUp` then waits for packet handler to change it to `netActive`; starts `serverTMTask` if this machine is server
- **Calls:** `NetBuildFirstRingPacket`, `NetSendRingPacket`, `machine_tick_count`, `alert_user`
- **Notes:** Blocks main thread in a loop waiting for `netActive`; packet handler transitions the state asynchronously

### RingGameProtocol::GetNetTime
- **Signature:** `int32 GetNetTime()`
- **Purpose:** Return game engine's view of network time, adjusted for latency buffering
- **Outputs/Return:** Current local net time minus adaptive latency (or minus fixed offsets in non-adaptive mode)
- **Side effects:** None
- **Notes:** Only function where `sCurrentAdaptiveLatency` affects engine behavior; return value controls how far ahead the game engine looks

### process_packet_buffer_flags (static)
- **Signature:** `static void process_packet_buffer_flags(void *buffer, long buffer_size, short packet_tag)`
- **Purpose:** Main packet processing: extract flags from unpacked ring packet, call `process_flags`, then forward packet upring
- **Inputs:** 
  - `buffer` ΓÇô unpacked `NetPacket` structure
  - `buffer_size` ΓÇô size in bytes
  - `packet_tag` ΓÇô special tag if player took over server role (NONE otherwise)
- **Side effects:** 
  - If server and have enough data: builds new ring packet, sends ACK, forwards upring
  - If server but not ready: copies packet to `status->buffer` for `NetServerTask` to send later
  - Updates adaptive latency state
- **Calls:** `process_flags`, `NetAddFlagsToPacket`, `NetBuildRingPacket`, `NetRebuildRingPacket`, `NetSendAcknowledgement`, `NetSendRingPacket`, `update_adaptive_latency`, `memcpy`

### process_flags (static)
- **Signature:** `static void process_flags(NetPacket* packet_data)`
- **Purpose:** Distribute action flags from received ring packet to each player's input queue
- **Inputs:** Unpacked `NetPacket` with `action_flags[]` array and per-player counts
- **Side effects:** Calls `process_action_flags()` (from interface.h) for each player; records stats
- **Calls:** `process_action_flags`
- **Notes:** Handles NET_DEAD players by stuffing zero flags instead

### drop_upring_player (static)
- **Signature:** `static void drop_upring_player(void)`
- **Purpose:** Remove dead upring neighbor from ring; promote self to server if upring was server
- **Side effects:** 
  - Removes upring player's action flags from ring packet
  - Adjusts upring address to skip the dead player
  - If upring was server: sets self as new server, swaps `queueingTMTask` Γåö `serverTMTask`
  - Sends modified ring packet with tag `tagCHANGE_RING_PACKET`
- **Calls:** `NetAdjustUpringAddressUpwards`, `myTMRemove`, `myXTMSetup`, `NetBuildRingPacket`, `NetRebuildRingPacket`, `memmove`
- **Notes:** Triggered by `NetCheckResendRingPacket()` after `kRETRIES` timeouts (Γëê50 ├ù 40ms)

### update_adaptive_latency (static, conditional compile)
- **Signature:** `static void update_adaptive_latency(int measurement = 0, int tick = 0)` (three variants: _LATENCY, _2, _3)
- **Purpose:** Adjust `sCurrentAdaptiveLatency` based on ring packet latency measurements
- **Inputs:** 
  - `measurement` ΓÇô latest measured ring latency (used by LATENCY_2, _3)
  - `tick` ΓÇô current game tick (used by LATENCY_3)
- **Side effects:** May increment/decrement `sCurrentAdaptiveLatency` and log changes
- **Calls:** `logNote1`, `logDump2`
- **Notes:** 
  - LATENCY: window-based averaging (not used)
  - LATENCY_2: compare average latency to thresholds, adjust smoothly
  - LATENCY_3 (enabled): track peak latency; decay to second-peak if timeout exceeded

## Control Flow Notes

**Initialization phase (Sync ΓåÆ netActive):**
1. Server calls `NetBuildFirstRingPacket()` and `NetSendRingPacket()` (triggers ring)
2. Non-servers start `NetQueueingTask` (periodic, captures input into `local_queue`)
3. Server starts `NetServerTask` (periodic, sends queued flags as new ring packets)
4. Each player's packet handler processes incoming ring, calls `process_packet_buffer_flags()` to forward
5. After ring completes two cycles, `netState` changes to `netActive` (in packet handler)

**Active frame loop:**
- **Every game tick:** Game engine calls `GetNetTime()` and dequeues action flags from player queues
- **NetQueueingTask** (Γëê10ms period): Captures local input, enqueues to `local_queue`
- **NetServerTask** (Γëê100ms period): Sends ring packet if `clearToForwardRing` is set by packet handler
- **Packet handler** (asynchronous): On ring arrival, calls `process_packet_buffer_flags()` to forward and trigger server send
- **ResendTask** (per packet): If ACK not received in `kACK_TIMEOUT` (40ms), call `NetCheckResendRingPacket()` to retransmit or drop player

**Shutdown phase (UnSync ΓåÆ netDown):**
1. Server sets `netState` to `netComingDown`; sends final packet with `required_action_flags = 0` or unsync type
2. Packet handler stops forwarding; players stop queueing
3. Tasks removed in `Exit1()`, memory freed in `Exit2()`

## External Dependencies
- **ActionQueues.h:** `GetRealActionQueues()` for accessing local player's action queue
- **player.h:** Player constants and queue accessors
- **network.h, network_private.h:** Topology, packet headers, state constants
- **network_data_formats.h:** `netcpy()` for pack/unpack; `SIZEOF_NetPacket`, etc.
- **mytm.h:** `myTMSetup`, `myXTMSetup`, `myTMRemove` for background task scheduling
- **map.h:** `TICKS_PER_SECOND` constant
- **vbl.h:** `parse_keymap` (called from elsewhere)
- **interface.h:** `process_action_flags()`
- **XML_ElementParser.h:** XML config parsing base class
- **Logging.h:** `logNote1`, `logDump2`, `logWarning2`
- **SDL_net (via sdl_network.h):** DDP networking (frames, sockets)
