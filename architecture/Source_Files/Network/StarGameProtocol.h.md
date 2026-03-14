# Source_Files/Network/StarGameProtocol.h

## File Purpose
Header file defining `StarGameProtocol`, a concrete implementation of the `NetworkGameProtocol` interface for star-topology network game protocol handling. Provides the contract for initialization, packet handling, state synchronization, and information distribution in a star-topology multiplayer game.

## Core Responsibilities
- Implement the `NetworkGameProtocol` virtual interface for star-topology networking
- Initialize and shut down the protocol layer (two-phase exit)
- Distribute game information/events to all players with optional filtering
- Synchronize and desynchronize game state across the network
- Handle incoming network packets and route them appropriately
- Manage network clock/timing
- Track unconfirmed action flags for client-side prediction
- Parse protocol configuration via XML

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| StarGameProtocol | class | Concrete implementation of NetworkGameProtocol for star topology |

## Global / File-Static State
None (state is instance-based within `StarGameProtocol`; external preference functions are available).

## Key Functions / Methods

### Enter
- Signature: `bool Enter(short* inNetStatePtr)`
- Purpose: Initialize the star protocol; begin network game session
- Inputs: Pointer to network state variable
- Outputs/Return: Success/failure boolean
- Side effects: Initializes protocol state, likely sets up listeners or timers
- Calls: Not inferable from this file
- Notes: Virtual override; inNetStatePtr allows caller to track state externally

### Exit1 / Exit2
- Signature: `void Exit1()`, `void Exit2()`
- Purpose: Two-phase shutdown of the protocol (first phase: graceful cleanup; second phase: final teardown)
- Inputs: None
- Outputs/Return: None
- Side effects: Release resources, close connections
- Calls: Not inferable from this file
- Notes: Pattern suggests ordered cleanup (e.g., notify peers, then release memory)

### DistributeInformation
- Signature: `void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team)`
- Purpose: Broadcast game information (events, state updates) to other players with filtering options
- Inputs: `type` (message type), `buffer` (payload), `buffer_size` (bytes), `send_to_self` (include sender), `only_send_to_team` (filter by team)
- Outputs/Return: None
- Side effects: Network transmission; may update local state
- Calls: Not inferable from this file
- Notes: Generic void* suggests this carries arbitrary game events

### Sync / UnSync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`, `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Synchronize game state with the network; desynchronize on end of game
- Inputs: (Sync) topology, smallest game tick, player indices; (UnSync) graceful flag, postgame tick
- Outputs/Return: Success boolean
- Side effects: Align local state with network or prepare for postgame
- Calls: Not inferable from this file
- Notes: `inSmallestGameTick` suggests tick-based game loop; `inGraceful` implies different shutdown modes

### GetNetTime
- Signature: `int32 GetNetTime()`
- Purpose: Retrieve current network time
- Inputs: None
- Outputs/Return: Network time (int32)
- Side effects: None
- Calls: Not inferable from this file
- Notes: Used for synchronizing ticks and client-server clock alignment

### PacketHandler
- Signature: `void PacketHandler(DDPPacketBuffer* inPacket)`
- Purpose: Process incoming network packets from peers
- Inputs: Packet buffer
- Outputs/Return: None
- Side effects: Parse packet, update state, dispatch events
- Calls: Not inferable from this file
- Notes: Entry point for all incoming messages in star topology

### GetUnconfirmedActionFlagsCount / PeekUnconfirmedActionFlag / UpdateUnconfirmedActionFlags
- Signature: `int32 GetUnconfirmedActionFlagsCount()`, `uint32 PeekUnconfirmedActionFlag(int32 offset)`, `void UpdateUnconfirmedActionFlags()`
- Purpose: Manage unconfirmed player actions for client-side prediction; allows peeking at predicted actions before server confirmation
- Inputs: (Peek) offset into unconfirmed queue
- Outputs/Return: (Count) action count, (Peek) action flag at offset
- Side effects: (Update) refreshes prediction queue
- Calls: Not inferable from this file
- Notes: Essential for reducing perceived latency in multiplayer games

### GetParser (static)
- Signature: `static XML_ElementParser* GetParser()`
- Purpose: Factory method to obtain XML parser for protocol configuration
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser
- Side effects: None (or allocates parser on first call)
- Calls: Not inferable from this file

## Control Flow Notes
Typical lifecycle:
1. **Init**: `Enter()` to establish protocol
2. **Frame**: `PacketHandler()` processes incoming packets; `DistributeInformation()` sends outgoing messages; `UpdateUnconfirmedActionFlags()` advances prediction
3. **Sync**: `Sync()` called at game start to align state; `UnSync()` at game end
4. **Shutdown**: `Exit1()` then `Exit2()` for ordered teardown

The `GetNetTime()` is polled to drive frame timing.

## External Dependencies
- **Includes**: `NetworkGameProtocol.h` (base class), `<stdio.h>`
- **Forward declarations**: `XML_ElementParser`
- **Symbols from other headers** (not defined here): `NetTopology`, `DDPPacketBuffer`
- **External functions**: `DefaultStarPreferences()`, `WriteStarPreferences(FILE*)` ΓÇö configure and persist star protocol preferences
