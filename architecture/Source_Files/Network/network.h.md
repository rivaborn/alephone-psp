# Source_Files/Network/network.h

## File Purpose
Public interface header for the Aleph One game engine's network subsystem. Defines types, callbacks, and function prototypes for multiplayer game gathering, joining, state synchronization, and in-game message distribution across network players.

## Core Responsibilities
- Define game and player information structures passed over network
- Manage network state machine (gathering ΓåÆ joining ΓåÆ waiting ΓåÆ active ΓåÆ shutdown)
- Provide game gathering (host side) and joining (client side) mechanisms
- Handle player topology setup and synchronization
- Distribute game data, updates, and chat messages across network
- Manage player connections, disconnections, and color/team changes
- Measure and track network latency for prediction
- Enforce cheat restrictions based on network mode

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `game_info` | struct | Game configuration: seed, type, time/kill limits, difficulty, level, network parameters |
| `player_info` | struct | Player metadata: name, color, team, serial number |
| `prospective_joiner_info` | struct | Joining player state: stream_id, name, color, team, gathering flag |
| `GatherCallbacks` | abstract class | Virtual methods for gather phase events (join success, player drops, changes) |
| `ChatCallbacks` | abstract class | Virtual method for chat message reception; static method for sending |
| `InGameChatCallbacks` | class | Singleton implementation of ChatCallbacks for in-game chat |

## Global / File-Static State
None (header-only; no static storage defined here).

## Key Functions / Methods

### NetEnter / NetExit
- **Purpose**: Lifecycle entry/exit for network subsystem.
- **Returns**: `bool` (NetEnter success status).

### NetGather / NetGatherPlayer
- **Purpose**: Host initiates gathering; accepts incoming joiner requests.
- **NetGather signature**: `bool NetGather(void *game_data, short game_data_size, void *player_data, short player_data_size, bool resuming_game)`
- **NetGatherPlayer signature**: `int NetGatherPlayer(const prospective_joiner_info &player, CheckPlayerProcPtr check_player)`
- **Returns**: bool for NetGather; int enum (kGatherPlayerFailed, kGatherPlayerSuccessful, kGatheredUnacceptablePlayer) for NetGatherPlayer.
- **Notes**: Game and player data passed as opaque buffers; caller-defined check procedure validates joiner.

### NetGameJoin
- **Purpose**: Client joins a game hosted at specified address.
- **Inputs**: player_data (opaque), player_data_size, host_address_string.
- **Returns**: `bool` success.

### NetUpdateJoinState
- **Purpose**: Polls join progress; returns pseudo-states (netPlayerAdded, netChatMessageReceived, netStartingResumeGame, etc.).
- **Returns**: `short` state enum value.
- **Notes**: Pseudo-states return but don't update internal NetState(); see comments.

### NetStart / NetSync / NetUnSync
- **Purpose**: Synchronize all players before game start; begin/end synchronized command execution.
- **Returns**: `bool` success.

### NetDistributeInformation / NetAddDistributionFunction
- **Purpose**: Route typed messages to remote players; register handlers for message types.
- **Inputs**: type (int16), buffer, size, send_to_self flag, optional send_only_to_team flag.
- **Notes**: Uses STL map; unknown message types (when received) are passed but ignored.

### NetGetMostRecentChatMessage
- **Purpose**: Retrieve pending chat before/after game.
- **Returns**: `bool` (true if message pending); outputs player_info* and message string pointer.
- **Notes**: Pointers valid only until next NetUpdateJoinState or NetCheckForIncomingMessages.

### NetGetLatency / NetDisplayPings
- **Purpose**: Query network latency (in ms) for prediction; check if HUD should show pings instead of scores.
- **Returns**: `int32` (latency) or kNetLatencyInvalid / kNetLatencyDisconnected; `bool` for display flag.
- **Notes**: Latency queries only work when caller is the hub.

### NetGetUnconfirmedActionFlags
- **Purpose**: Retrieve predicted action flags for client-side prediction.
- **Returns**: Count and individual flags (uint32) by offset.

### Cheat restriction checks
- **NetAllowCrosshair / NetAllowTunnelVision / NetAllowBehindview / NetAllowCarnageMessages / NetAllowSavingLevel**: Return `bool`; disable features in network mode.

## Control Flow Notes
Not fully inferable from header alone (implementation in network.c), but structure implies:
1. **Gather phase**: Host calls `NetEnter()`, then `NetGather()` to open session; `NetGatherPlayer()` for each joiner acceptance.
2. **Join phase**: Client calls `NetEnter()`, then `NetGameJoin(host_address)` and polls `NetUpdateJoinState()`.
3. **Sync phase**: All players call `NetSync()`, then `NetStart()` when ready.
4. **Active phase**: `NetProcessMessagesInGame()` dispatches in-game updates; `NetDistributeInformation()` sends typed messages.
5. **Shutdown**: `NetExit()` closes session; `NetCancelGather()` / `NetCancelJoin()` for early termination.

## External Dependencies
- **cseries.h, cstypes.h**: Portability macros and cross-platform types (int16, uint16, int32, uint32, byte, OSErr).
- **C++ STL**: `std::string` used in chat callbacks.
- **Forward references** (defined elsewhere): `entry_point`, `player_start_data`, `SSLP_ServiceInstance`.
- **Callback interface pattern**: GatherCallbacks, ChatCallbacks established but invoked by network.c (not shown).
