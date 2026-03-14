# Source_Files/Network/StarGameProtocol.cpp

## File Purpose

Glue layer bridging the star-topology network protocol implementation to the Aleph One game engine. Manages network synchronization, packet routing, and action flag distribution for hub-and-spoke multiplayer games.

## Core Responsibilities

- Adapt legacy action queues to tick-based circular queue interface
- Manage network lifecycle (initialization, synchronization, cleanup)
- Route incoming packets to hub or spoke handlers based on network role
- Distribute game information with lossy streaming support
- Track unconfirmed action flags and network timing
- Provide XML configuration parsing for network preferences

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LegacyActionQueueToTickBasedQueueAdapter<tValueType>` | Template class | Adapts legacy action queue to `WritableTickBasedCircularQueue` interface for single-player index |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sStarQueues` | `WritableTickBasedActionQueue*[]` | static | Per-player action queues in star topology (max `MAXIMUM_NUMBER_OF_NETWORK_PLAYERS`) |
| `sHubIsLocal` | `bool` | static | Whether local machine is the hub/server node |
| `sTopology` | `NetTopology*` | static | Current network topology (players, addresses, state) |
| `sNetStatePtr` | `short*` | static | Pointer to engine's network state variable |
| `sStarParser` | `XML_ElementParser` | static | XML parser for star protocol configuration (lazy-init) |

## Key Functions / Methods

### StarGameProtocol::Sync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- Purpose: Initialize hub/spoke components and synchronize network game
- Inputs: Topology with player list, smallest tick, local index, server index
- Outputs/Return: Always `true`
- Side effects: Creates action queue adapters, initializes hub (if local is server), initializes spoke, sets network state to `netActive`
- Calls: `hub_initialize()`, `spoke_initialize()`
- Notes: Invokes hub *and* spoke; spoke handles local player regardless of role

### StarGameProtocol::UnSync
- Signature: `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Clean up network modules and deallocate action queues
- Inputs: Graceful shutdown flag, postgame tick threshold
- Outputs/Return: Always `true`
- Side effects: Calls `spoke_cleanup()`, `hub_cleanup()` (if hub was local), deletes action queue adapters, sets state to `netDown`
- Calls: `spoke_cleanup()`, `hub_cleanup()`
- Notes: Only acts if state is `netStartingUp` or `netActive`

### StarGameProtocol::PacketHandler
- Signature: `void PacketHandler(DDPPacketBufferPtr packet)`
- Purpose: Route incoming DDP packet to hub or spoke handler
- Inputs: Packet buffer
- Outputs/Return: None
- Side effects: Delivers packet to `hub_received_network_packet()` or `spoke_received_network_packet()` based on `sHubIsLocal`
- Calls: Hub or spoke handler
- Notes: Simple dispatcher

### StarGameProtocol::DistributeInformation
- Signature: `void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team)`
- Purpose: Send game information to other players, with lossy-streaming support
- Inputs: Distribution type, data buffer, size, flags
- Outputs/Return: None
- Side effects: Transmits via `spoke_distribute_lossy_streaming_bytes_to_everyone()` if distribution type supports lossy
- Calls: `NetGetDistributionInfoForType()`, `spoke_distribute_lossy_streaming_bytes_to_everyone()`
- Notes: Only distributes if lossy flag is set in distribution info

### StarGameProtocol::GetUnconfirmedActionFlagsCount
- Signature: `int32 GetUnconfirmedActionFlagsCount()`
- Purpose: Return pending unconfirmed action flags
- Inputs: None
- Outputs/Return: Write tick minus read tick
- Side effects: None
- Calls: `spoke_get_unconfirmed_flags_queue()`

### StarGameProtocol::UpdateUnconfirmedActionFlags
- Signature: `void UpdateUnconfirmedActionFlags()`
- Purpose: Advance read pointer as flags become confirmed
- Inputs: None
- Outputs/Return: None
- Side effects: Dequeues confirmed flags while respecting bounds
- Calls: `spoke_get_unconfirmed_flags_queue()`, `spoke_get_smallest_unconfirmed_tick()`

**Other trivial methods:** `Enter()` (store state ptr), `Exit1()`/`Exit2()` (empty), `GetNetTime()` (pass-through), `PeekUnconfirmedActionFlag()` (peek queue), `GetParser()` (lazy XML parser init).

**Utility functions:** `make_player_really_net_dead()` (mark player disconnected), `call_distribution_response_function_if_available()` (invoke callback if registered), `DefaultStarPreferences()`/`WriteStarPreferences()` (config management).

## Control Flow Notes

1. **Network startup:** `Enter()` ΓåÆ `Sync()` (initializes hub if server, always spoke)  
2. **Active play:** `PacketHandler()` dispatches packets; action flags enqueued via `LegacyActionQueueToTickBasedQueueAdapter`; unconfirmed flags tracked and advanced  
3. **Shutdown:** `UnSync()` cleans spoke/hub; `Exit1()`/`Exit2()` (unused)  

The adapter pattern (`LegacyActionQueueToTickBasedQueueAdapter`) translates legacy `process_action_flags()` calls into tick-based queue operations, enabling backward compatibility with existing action queue code.

## External Dependencies

- `network_star.h` ΓÇö Hub/spoke protocol functions (`hub_initialize`, `spoke_initialize`, `spoke_get_net_time`, etc.)
- `TickBasedCircularQueue.h` ΓÇö `WritableTickBasedCircularQueue<action_flags_t>`
- `player.h` ΓÇö `GetRealActionQueues()`
- `interface.h` ΓÇö `process_action_flags()` (used in adapter)
- Base class `NetworkGameProtocol` (interface contract)
- Defined elsewhere: `NetTopology`, `DDPPacketBuffer`, `NetDistributionInfo`, `NetAddrBlock`
