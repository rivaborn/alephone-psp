# Source_Files/Network/NetworkGameProtocol.cpp

## File Purpose
Implementation file for the NetworkGameProtocol abstract base class. Currently contains only includes and no implementation; the interface is defined in the corresponding header and serves as the protocol contract for network game communication in the Aleph One game engine.

## Core Responsibilities
- Define and export the `NetworkGameProtocol` abstract interface for network game protocol implementations
- Establish the contract for game state synchronization across the network
- Define the lifecycle for network game sessions (Enter, Exit1, Exit2)
- Specify packet handling and information distribution mechanisms
- Manage unconfirmed action flags for client-side prediction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NetworkGameProtocol` | abstract class | Pure virtual interface defining the contract for network game protocol implementations |

## Global / File-Static State
None.

## Key Functions / Methods

### Enter
- Signature: `virtual bool Enter(short* inNetStatePtr) = 0`
- Purpose: Initialize the network game protocol session
- Inputs: Pointer to network state
- Outputs/Return: Boolean success/failure status
- Notes: Called at session startup; pure virtual (implementation in derived classes)

### Exit1 / Exit2
- Signature: `virtual void Exit1() = 0` and `virtual void Exit2() = 0`
- Purpose: Two-phase shutdown of the network session
- Notes: Likely for graceful disconnection; pure virtual

### DistributeInformation
- Signature: `virtual void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team) = 0`
- Purpose: Broadcast game state or event information across the network
- Inputs: Message type, data buffer, size, and routing flags (send to self, team only)
- Notes: Core message distribution mechanism

### Sync / UnSync
- Signature: `virtual bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex) = 0` and `virtual bool UnSync(bool inGraceful, int32 inSmallestPostgameTick) = 0`
- Purpose: Establish network synchronization state; gracefully desynchronize
- Notes: Sync takes network topology and player indices; UnSync supports graceful or abrupt modes

### GetNetTime
- Signature: `virtual int32 GetNetTime() = 0`
- Purpose: Retrieve the current network game tick or time reference
- Outputs/Return: 32-bit integer time value

### PacketHandler
- Signature: `virtual void PacketHandler(DDPPacketBuffer* inPacket) = 0`
- Purpose: Process incoming network packets
- Inputs: DDP (Datagram Delivery Protocol) packet buffer
- Notes: Entry point for packet reception and processing

### Unconfirmed Action Flags
- `GetUnconfirmedActionFlagsCount()`, `PeekUnconfirmedActionFlag(int32 offset)`, `UpdateUnconfirmedActionFlags()`
- Purpose: Manage client-side predicted actions pending server confirmation
- Notes: Supports optimistic client-side prediction without server round-trip confirmation

## Control Flow Notes
This is a frame-based network protocol interface. During initialization: `Enter()` ΓåÆ `Sync()`. During gameplay: `PacketHandler()` receives packets; `DistributeInformation()` broadcasts state. At shutdown: `UnSync()` ΓåÆ `Exit1()` / `Exit2()`. The unconfirmed action flags suggest a client-prediction model where clients predict actions locally while awaiting authoritative confirmation from the server.

## External Dependencies
- `network_private.h` ΓÇô internal network module definitions
- `DDPPacketBuffer`, `NetTopology` ΓÇô types defined elsewhere (DDP packet and network topology)
- Pure virtual interface; all methods implemented by derived classes
