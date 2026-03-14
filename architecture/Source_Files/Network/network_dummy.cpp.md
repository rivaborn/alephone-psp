# Source_Files/Network/network_dummy.cpp

## File Purpose
Provides stub/dummy implementations of all network interface functions for single-player or network-disabled builds. All functions return trivial values or perform no operations, allowing the game to compile and run without network support. This is a fallback implementation that satisfies the linker when actual network code is unavailable.

## Core Responsibilities
- Provide no-op implementations of all network API functions declared in `network.h`
- Enable compilation when real network subsystem is disabled or unavailable
- Allow single-player game operation by returning safe default values
- Stub all player query, synchronization, and map change functions
- Disable network-dependent features (crosshair, tunnel vision, behind-view in multiplayer)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### NetExit
- Signature: `void NetExit(void)`
- Purpose: Dummy shutdown for network subsystem.
- Inputs: None.
- Outputs/Return: None.
- Side effects: None.
- Calls: None.
- Notes: No-op.

### NetSync / NetUnSync
- Signature: `bool NetSync(void)`, `bool NetUnSync(void)`
- Purpose: Dummy synchronization points for network protocol.
- Inputs: None.
- Outputs/Return: `true` (always synchronized in single-player).
- Side effects: None.
- Calls: None.
- Notes: Return `true` to indicate the dummy network is "in sync."

### NetGetNumberOfPlayers
- Signature: `short NetGetNumberOfPlayers(void)`
- Purpose: Report player count for single-player game.
- Inputs: None.
- Outputs/Return: `1` (always single player).
- Side effects: None.
- Calls: None.
- Notes: Hardcoded to 1 to simulate single-player.

### NetGetLocalPlayerIndex
- Signature: `short NetGetLocalPlayerIndex(void)`
- Purpose: Return the local player index.
- Inputs: None.
- Outputs/Return: `0` (always player 0).
- Side effects: None.
- Calls: None.
- Notes: Hardcoded to 0.

### NetGetPlayerIdentifier
- Signature: `short NetGetPlayerIdentifier(short player_index)`
- Purpose: Look up player ID by index.
- Inputs: `player_index`.
- Outputs/Return: `0`.
- Side effects: None.
- Calls: None.
- Notes: Ignores input; always returns 0.

### NetGetPlayerData / NetGetGameData
- Signature: `void *NetGetPlayerData(short player_index)`, `void *NetGetGameData(void)`
- Purpose: Query player/game configuration data.
- Inputs: Optional `player_index`.
- Outputs/Return: `NULL`.
- Side effects: None.
- Calls: None.
- Notes: No data available in dummy mode.

### NetChangeMap
- Signature: `bool NetChangeMap(struct entry_point *entry)`
- Purpose: Attempt to change level across network.
- Inputs: `entry` (map entry point).
- Outputs/Return: `false` (always fails).
- Side effects: None.
- Calls: None.
- Notes: Network maps not supported.

### NetGetNetTime
- Signature: `long NetGetNetTime(void)`
- Purpose: Query elapsed network time.
- Inputs: None.
- Outputs/Return: `0`.
- Side effects: None.
- Calls: None.
- Notes: No network clock.

### network_gather / network_join
- Signature: `bool network_gather(void)`, `int network_join(void)`
- Purpose: Dummy multiplayer session setup.
- Inputs: None.
- Outputs/Return: `false` (always fail).
- Side effects: None.
- Calls: None.
- Notes: Multiplayer lobby not available.

### NetAllowCrosshair / NetAllowTunnelVision / NetAllowBehindview
- Signature: `bool NetAllow*(void)`
- Purpose: Query which cheats/features are enabled in network.
- Inputs: None.
- Outputs/Return: `false` (all disabled).
- Side effects: None.
- Calls: None.
- Notes: Conservative defaults; could be `true` in actual implementation.

### current_game_has_balls
- Signature: `bool current_game_has_balls(void)`
- Purpose: Query if game is Capture-the-Ball variant.
- Inputs: None.
- Outputs/Return: `false`.
- Side effects: None.
- Calls: None.
- Notes: Only meaningful in network multiplayer.

## Control Flow Notes
This file provides no active control flow. Functions are called by the game engine when it needs to query network state or coordinate multiplayer actions. In dummy mode, all queries return safe, neutral defaults (`NULL`, `false`, `0`, `1`) that allow single-player operation to proceed. The engine is expected to check return values and avoid network-dependent code paths when running with this dummy implementation.

## External Dependencies
- **Includes**: `cseries.h` (base types, compiler macros), `map.h` (struct `entry_point`), `network.h` (function declarations), `network_games.h` (presumably for game-mode queries)
- **External symbols**: All function names declared in `network.h`; `entry_point` struct defined in `map.h`
