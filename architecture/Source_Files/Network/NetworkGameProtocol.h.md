# Source_Files/Network/NetworkGameProtocol.h

## File Purpose
Abstract base class (interface) defining the contract for a generic network game protocol implementation. Decouples game networking logic from specific protocol implementations (e.g., ring protocol). Part of Aleph One's multi-player networking subsystem.

## Core Responsibilities
- **State management**: Enter/exit game network sessions with initialization and cleanup
- **Information distribution**: Send game/player data across the network with loss-model control (lossy vs. lossless)
- **Synchronization**: Establish and break network game synchronization across peers
- **Packet handling**: Process incoming network packets dispatched from the transport layer
- **Time coordination**: Provide authoritative network time for game frame synchronization
- **Action prediction**: Manage unconfirmed action flags for client-side prediction before network acknowledgment

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### Enter
- **Signature:** `virtual bool Enter(short* inNetStatePtr) = 0`
- **Purpose:** Initialize and join the game network session.
- **Inputs:** Pointer to network state variable.
- **Outputs/Return:** Boolean success/failure.
- **Side effects:** Initializes protocol state, modifies `*inNetStatePtr`.
- **Calls:** Not inferable from this file.
- **Notes:** Called during game setup phase before game loop begins.

### Exit1 / Exit2
- **Signature:** `virtual void Exit1() = 0`, `virtual void Exit2() = 0`
- **Purpose:** Two-phase shutdown of the game network session (graceful cleanup).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Tears down protocol state.
- **Calls:** Not inferable from this file.
- **Notes:** Two phases likely allow flushing pending packets before final disconnect.

### DistributeInformation
- **Signature:** `virtual void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team) = 0`
- **Purpose:** Broadcast game/player data to network peers with optional filtering.
- **Inputs:** `type` (distribution type), `buffer` (data payload), `buffer_size`, `send_to_self`, `only_send_to_team`.
- **Outputs/Return:** None.
- **Side effects:** Transmits network packet(s).
- **Calls:** Not inferable from this file.
- **Notes:** Supports both team-only and full-broadcast modes.

### Sync / UnSync
- **Signature:** `virtual bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex) = 0`, `virtual bool UnSync(bool inGraceful, int32 inSmallestPostgameTick) = 0`
- **Purpose:** Establish synchronized game state across peers; break synchronization at session end.
- **Inputs:** `Sync`: topology snapshot, game tick floor, local/server player indices. `UnSync`: graceful flag, postgame tick floor.
- **Outputs/Return:** Boolean success/failure.
- **Side effects:** Waits for peer acknowledgment; modifies local network state.
- **Calls:** Not inferable from this file.
- **Notes:** `Sync` is part of game initialization; `UnSync` is part of shutdown.

### GetNetTime
- **Signature:** `virtual int32 GetNetTime() = 0`
- **Purpose:** Query the authoritative network clock.
- **Inputs:** None.
- **Outputs/Return:** Network time (int32, likely milliseconds or game ticks).
- **Side effects:** None (query-only).
- **Calls:** Not inferable from this file.
- **Notes:** Used for frame-lock and timestamp coordination.

### PacketHandler
- **Signature:** `virtual void PacketHandler(DDPPacketBuffer* inPacket) = 0`
- **Purpose:** Dispatch incoming network packet for protocol-specific processing.
- **Inputs:** Raw packet buffer from transport layer.
- **Outputs/Return:** None.
- **Side effects:** Updates internal protocol state, may trigger game events.
- **Calls:** Not inferable from this file.
- **Notes:** Called per incoming datagram during game loop.

### GetUnconfirmedActionFlagsCount / PeekUnconfirmedActionFlag / UpdateUnconfirmedActionFlags
- **Signature:** `virtual int32 GetUnconfirmedActionFlagsCount() = 0`, `virtual uint32 PeekUnconfirmedActionFlag(int32 offset) = 0`, `virtual void UpdateUnconfirmedActionFlags() = 0`
- **Purpose:** Manage local action flags pending server/peer acknowledgment for client-side prediction.
- **Inputs:** `PeekUnconfirmedActionFlag`: index offset into flag queue.
- **Outputs/Return:** Count (int32), flag value (uint32), or void.
- **Side effects:** `UpdateUnconfirmedActionFlags` modifies queue state.
- **Calls:** Not inferable from this file.
- **Notes:** These are read/write helpers for lookahead prediction without blocking on network confirmation.

## Control Flow Notes
This interface sits at the center of the networking loop:

1. **Session Init**: `Enter()` ΓåÆ `Sync()` establishes connection and synchronization.
2. **Game Loop**: `PacketHandler()` dispatched per packet; `DistributeInformation()` sends local updates; `GetNetTime()` queries clock; unconfirmed action flags queried for prediction.
3. **Session Shutdown**: `UnSync()` ΓåÆ `Exit1()` / `Exit2()` tears down cleanly.

## External Dependencies
- **Included:** `network_private.h` (provides `DDPPacketBuffer`, `NetTopology` types; also includes `cstypes.h`, `sdl_network.h`, `network.h`).
- **External symbols used:** `DDPPacketBuffer`, `NetTopology` (structs defined in network_private.h).
