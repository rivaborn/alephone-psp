# Source_Files/Network/RingGameProtocol.h

## File Purpose
Header file defining the RingGameProtocol class, which implements the NetworkGameProtocol interface for ring-topology multiplayer networking in Aleph One. Provides abstraction for synchronization, packet handling, and information distribution in network games using a ring network topology.

## Core Responsibilities
- Implement NetworkGameProtocol abstract interface for ring topology
- Manage network game initialization (Enter) and cleanup (Exit1/Exit2)
- Handle synchronization of network state and game ticks
- Route and distribute game information across the network
- Process incoming network packets
- Track and manage unconfirmed action flags for client-side prediction
- Provide static factory method for XML configuration parsing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| RingGameProtocol | class | Concrete implementation of NetworkGameProtocol for ring-topology networks |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| DefaultRingPreferences | extern function | global | Initialize default ring protocol preferences |
| WriteRingPreferences | extern function | global | Serialize ring protocol preferences to file |

## Key Functions / Methods

### Enter
- Signature: `bool Enter(short* inNetStatePtr)`
- Purpose: Initialize ring protocol and enter network game state
- Inputs: Pointer to network state variable
- Outputs/Return: Boolean success flag
- Side effects: Modifies network state, initiates protocol-specific setup
- Calls: (Not visible in header)
- Notes: Called during network game initialization; inNetStatePtr likely updated with current state

### Exit1 / Exit2
- Signature: `void Exit1()` / `void Exit2()`
- Purpose: Two-phase shutdown of ring protocol
- Inputs: None
- Outputs/Return: None
- Side effects: Cleans up protocol resources; Exit1 likely initiates graceful shutdown, Exit2 finalizes
- Calls: (Not visible in header)
- Notes: Two-phase design allows coordinated network shutdown

### DistributeInformation
- Signature: `void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team)`
- Purpose: Broadcast game information across the network
- Inputs: Message type, buffer data, buffer size, self-inclusion flag, team-only flag
- Outputs/Return: None (void)
- Side effects: Network transmission; may buffer locally if send_to_self=true
- Calls: (Not visible in header)
- Notes: Flexible routing via send_to_self and only_send_to_team flags

### Sync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- Purpose: Synchronize game state across the network
- Inputs: Network topology, smallest valid game tick, local and server player indices
- Outputs/Return: Boolean success flag
- Side effects: Exchanges game state; may block or queue sync operations
- Calls: (Not visible in header)
- Notes: Called periodically during game loop; indices identify local player and server

### UnSync
- Signature: `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Desynchronize and prepare for game shutdown
- Inputs: Graceful shutdown flag, smallest postgame tick
- Outputs/Return: Boolean success flag
- Side effects: Cleans up sync state; may log final game state
- Calls: (Not visible in header)

### GetNetTime
- Signature: `int32 GetNetTime()`
- Purpose: Retrieve current network-synchronized game time
- Inputs: None
- Outputs/Return: Current game tick/time as 32-bit integer
- Side effects: None (query-only)
- Calls: (Not visible in header)
- Notes: Used for synchronizing actions across network

### PacketHandler
- Signature: `void PacketHandler(DDPPacketBuffer* inPacket)`
- Purpose: Process incoming network packet
- Inputs: Pointer to received packet buffer
- Outputs/Return: None
- Side effects: Parses packet, updates protocol state, may trigger synchronization
- Calls: (Not visible in header)
- Notes: Called per received packet; implements ring protocol message parsing

### GetUnconfirmedActionFlagsCount / PeekUnconfirmedActionFlag / UpdateUnconfirmedActionFlags
- Signature: `int32 GetUnconfirmedActionFlagsCount()` / `uint32 PeekUnconfirmedActionFlag(int32 offset)` / `void UpdateUnconfirmedActionFlags()`
- Purpose: Manage client prediction using locally-predicted but unconfirmed actions
- Inputs: Offset into unconfirmed flag queue
- Outputs/Return: Count or flag value; UpdateUnconfirmedActionFlags returns void
- Side effects: UpdateUnconfirmedActionFlags advances prediction state
- Calls: (Not visible in header)
- Notes: Enables client-side prediction while awaiting server confirmation

### GetParser (static)
- Signature: `static XML_ElementParser* GetParser()`
- Purpose: Factory method for XML configuration parser
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser instance
- Side effects: May allocate parser instance
- Calls: (Not visible in header)
- Notes: Used for deserializing ring protocol configuration from XML

## Control Flow Notes
RingGameProtocol integrates into the game's network lifetime:
1. **Init**: `Enter()` establishes connection and initializes state
2. **Frame**: `Sync()` called periodically; `PacketHandler()` processes inbound packets; `DistributeInformation()` sends updates
3. **Client Prediction**: `GetUnconfirmedActionFlagsCount()`, `PeekUnconfirmedActionFlag()`, `UpdateUnconfirmedActionFlags()` manage speculative action prediction
4. **Shutdown**: `UnSync()` prepares for disconnection; `Exit1()`/`Exit2()` finalize cleanup

## External Dependencies
- **Base class**: `NetworkGameProtocol` (abstract interface)
- **Type declarations** (defined elsewhere): `NetTopology`, `DDPPacketBuffer`, `XML_ElementParser`
- **Included**: `stdio.h` (C standard I/O, used by WriteRingPreferences)
- **Network internals**: Dependencies via included `NetworkGameProtocol.h` ΓåÆ `network_private.h`
