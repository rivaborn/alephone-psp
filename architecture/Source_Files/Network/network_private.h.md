# Source_Files/Network/network_private.h

## File Purpose
Private header defining internal structures, constants, and interfaces for the network subsystem. Contains packet definitions for ring-protocol communication, game topology structures, error codes, and service discovery management. Not intended for use outside the networking module.

## Core Responsibilities
- Define network packet structures (ring packets, distribution packets, player topology)
- Specify packet tags, types, and protocol constants for ring-protocol gameplay
- Enumerate error codes and network states for internal error handling
- Define game-specific data structures (gathering, chat, game info, player info)
- Provide service discovery interface classes (SSLP-based gatherer/joiner announcement)
- Supply distribution info lookup for custom game data types

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| NetPacketHeader | struct | Base header for all datagrams; contains tag and sequence number |
| NetPacket | struct | Ring packet carrying action flags, game time, and ring packet type |
| NetDistributionPacket | struct | Datagram for non-action-flag data (custom game data) |
| NetPlayer | struct | Player entry in topology; holds address, identifier, stream ID, status |
| NetTopology | struct | Complete game topology with all players and game state |
| NetChatMessage | struct | Chat message with sender identifier and text buffer |
| NetDistributionInfo | struct | Metadata for custom distribution type (lossy flag, callback) |
| gather_player_data | struct | Application-level: player identifier for gathering phase |
| accept_gather_data | struct | Application-level: gatherer's acceptance response with version negotiation |
| ClientChatInfo | struct | Chat metadata: player name, color, team |
| GathererAvailableAnnouncer | class | Manages SSLP service instance for local game availability |
| JoinerSeekingGathererAnnouncer | class | Manages SSLP service discovery; finds available gatherers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| GAME_PORT | #define (int) | global | UDP port for game traffic (4226) |
| asyncUncompleted | #define (int) | global | ioResult value for incomplete operations (1) |
| MAXIMUM_GAME_DATA_SIZE | #define | global | Max bytes for game-level custom data (256) |
| MAXIMUM_PLAYER_DATA_SIZE | #define | global | Max bytes for player-level custom data (128) |
| MAXIMUM_UPDATES_PER_PACKET | #define | global | Max action flags per player per ring packet (16) |
| UPDATE_LATENCY, UPDATES_PER_PACKET | #define | global | Ring protocol timing constants |
| NET_QUEUE_SIZE, UNSYNC_TIMEOUT, STREAM_TRANSFER_CHUNK_SIZE | #define | global | Network operation timeouts and buffer sizes |
| kACK_TIMEOUT, kRETRIES | #define | global | Retransmission parameters (40ms timeout, 50 retries before drop) |
| kPROTOCOL_TYPE | #define | global | Network version identifier (69) |
| tagRING_PACKET, tagACKNOWLEDGEMENT, ΓÇª | enum tags | global | Datagram type identifiers for packet routing |
| typeSYNC_RING_PACKET, typeTIME_RING_PACKET, ΓÇª | enum types | global | Ring packet classification (sync, time, normal, unsync, dead) |
| strNETWORK_ERRORS enum | enum codes | global | User-facing error strings (132 resource ID) |
| Stream packet types (_hello_packet, _joiner_info_packet, ΓÇª) | enum | global | Topology streaming packet types for setup phase |
| Version negotiation codes (kNaiveJoinerAccepted, kResumeNetgameSavvyJoinerAccepted, ΓÇª) | enum | global | Protocol version markers for compatibility detection |

## Key Functions / Methods

### NetGetDistributionInfoForType
- Signature: `const NetDistributionInfo* NetGetDistributionInfoForType(int16 inType);`
- Purpose: Look up lossy flag and callback procedure for a custom distribution type
- Inputs: `inType` ΓÇö 16-bit distribution type identifier
- Outputs/Return: Pointer to NetDistributionInfo (lossy, distribution_proc), or NULL if not found
- Side effects: None (query only)
- Calls: Not inferable from this file
- Notes: Supports arbitrary custom game data types via registration in network.cpp

### GathererAvailableAnnouncer (constructor)
- Signature: `GathererAvailableAnnouncer();`
- Purpose: Initialize SSLP service instance to advertise local game as available
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates and registers SSLP_ServiceInstance; starts advertising
- Calls: SSLP API (implicit in implementation)
- Notes: Single instance per gatherer; publishing stops on destruction

### GathererAvailableAnnouncer::pump
- Signature: `static void pump();`
- Purpose: Give SSLP processing time to send announcements (called periodically from main loop)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates SSLP state; may send multicast packets
- Calls: SSLP_Pump() (from implementation)
- Notes: Should be called frequently (ΓëÑ1/sec recommended per SSLP_API.h)

### JoinerSeekingGathererAnnouncer (constructor)
- Signature: `JoinerSeekingGathererAnnouncer(bool shouldSeek);`
- Purpose: Initialize SSLP service discovery to locate available gatherers
- Inputs: `shouldSeek` ΓÇö whether to start active searching immediately
- Outputs/Return: None
- Side effects: Registers found/lost callbacks; starts listening if shouldSeek=true
- Calls: SSLP_Locate_Service_Instances() or equivalent (implementation)
- Notes: Callbacks are static members; valid until destruction

### JoinerSeekingGathererAnnouncer::pump
- Signature: `static void pump();`
- Purpose: Give SSLP processing time for discovery updates (called periodically from main loop)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates SSLP state; may invoke found/lost callbacks
- Calls: SSLP_Pump() (from implementation)
- Notes: Pump must be called for any callback invocation to occur

### JoinerSeekingGathererAnnouncer::found_gatherer_callback
- Signature: `static void found_gatherer_callback(const SSLP_ServiceInstance* instance);`
- Purpose: Invoked when SSLP discovers an available gatherer
- Inputs: `instance` ΓÇö discovered service; valid until corresponding lost callback
- Outputs/Return: None
- Side effects: Likely notifies UI or triggers join attempt
- Calls: Not inferable (implementation in .cpp)
- Notes: May be called from different thread; should not block

### JoinerSeekingGathererAnnouncer::lost_gatherer_callback
- Signature: `static void lost_gatherer_callback(const SSLP_ServiceInstance* instance);`
- Purpose: Invoked when a previously discovered gatherer becomes unavailable
- Inputs: `instance` ΓÇö same pointer passed to found_gatherer_callback; invalidated after return
- Outputs/Return: None
- Side effects: Likely notifies UI; pointer invalid after return
- Calls: Not inferable
- Notes: Will not be called unless found callback was called for this instance

## Control Flow Notes

**Gathering Phase (Gatherer):**
- `GathererAvailableAnnouncer` is instantiated to make the game discoverable via SSLP.
- `pump()` is called regularly so multicast announcements reach joiners.
- Topology is built from `gather_player_data` and `accept_gather_data` messages via streaming protocol.

**Joining Phase (Joiner):**
- `JoinerSeekingGathererAnnouncer` searches for available games via SSLP.
- Discovered instances trigger `found_gatherer_callback`, allowing UI to present game list.
- Lost instances trigger `lost_gatherer_callback` to remove stale entries.
- Upon selection, joiner connects and sends topology streaming packets to gather.

**Active Gameplay:**
- Ring packets (`NetPacket`) flow continuously; tags distinguish ring-packet from distribution data.
- Action flags accumulate per player and are sent with specified latency/frequency (`UPDATES_PER_PACKET`).
- Distribution packets carry custom data (game state, chat, scripts) on separate channel.
- Chat messages (`NetChatMessage`) are encapsulated in streaming packets during pre-game and in-game phases.

**Shutdown:**
- `GathererAvailableAnnouncer` and `JoinerSeekingGathererAnnouncer` destructors clean up SSLP state.

## External Dependencies

- **cstypes.h** ΓÇö Fixed-width integer types (int8, int16, uint32, etc.), NONE constant
- **sdl_network.h** ΓÇö SDL_net types (IPaddress, UDPsocket, TCPsocket); game-port UDP transport
- **network.h** ΓÇö Public API; game_info, player_info, NetDistributionProc typedef, ChatCallbacks, GatherCallbacks
- **SSLP_API.h** ΓÇö Service discovery: SSLP_ServiceInstance, SSLP_Pump, SSLP_Allow_Service_Discovery, SSLP_Locate_Service_Instances
- **\<memory\>** ΓÇö std::string for ClientChatInfo
- **Forward declarations** ΓÇö CommunicationsChannel, MessageDispatcher, MessageHandler, Message (used in implementation, not defined here)
