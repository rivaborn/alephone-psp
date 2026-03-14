# Source_Files/Network/SDL_netx.h

## File Purpose
Header file declaring the public API for SDL_netx, a network extension library that adds broadcast capabilities to SDL_net. Provides functions to enable/disable UDP broadcast mode and send broadcast packets across all available network interfaces.

## Core Responsibilities
- Enable broadcast mode on UDP sockets
- Disable broadcast mode on UDP sockets
- Send UDP packets to all available broadcast addresses in the system
- Manage socket capabilities for broadcast transmission

## Key Types / Data Structures
None (uses SDL_net types: `UDPsocket`, `UDPpacket`).

## Global / File-Static State
None.

## Key Functions / Methods

### SDLNetx_EnableBroadcast
- Signature: `int SDLNetx_EnableBroadcast(UDPsocket inSocket)`
- Purpose: Enable broadcasting mode on a UDP socket, allowing subsequent calls to `SDLNetx_UDP_Broadcast` to succeed.
- Inputs: `inSocket` ΓÇô UDP socket to configure
- Outputs/Return: Positive integer if broadcasting enabled successfully; 0 if failed
- Side effects: Modifies socket options (SO_BROADCAST or equivalent platform socket flag)
- Notes: Must be called before attempting broadcast sends; implementation platform-dependent

### SDLNetx_DisableBroadcast
- Signature: `int SDLNetx_DisableBroadcast(UDPsocket inSocket)`
- Purpose: Disable broadcasting mode on a UDP socket, preventing broadcast sends.
- Inputs: `inSocket` ΓÇô UDP socket to configure
- Outputs/Return: Positive integer if broadcasting disabled successfully; 0 if failed
- Side effects: Modifies socket options
- Notes: Inverse of `EnableBroadcast`

### SDLNetx_UDP_Broadcast
- Signature: `int SDLNetx_UDP_Broadcast(UDPsocket inSocket, UDPpacket* inPacket)`
- Purpose: Send a packet to the broadcast address on all available network interfaces supporting UDP/IP.
- Inputs: 
  - `inSocket` ΓÇô socket configured with `EnableBroadcast()` (channel info ignored)
  - `inPacket` ΓÇô packet to broadcast; host-part of address ignored; port-part used
- Outputs/Return: Number of interfaces the packet was broadcast on (0 if not broadcast); `inPacket->status` contains result from last interface attempt
- Side effects: Sends network data; modifies `inPacket->status`
- Notes: Iterates over all available network interfaces; address handling peculiarity (host ignored, port used)

## Control Flow Notes
Not inferable from header. This is a socket utility library; functions are called from application code during network init/teardown phases and during frame updates when broadcast communication is needed.

## External Dependencies
- `SDL_net.h` ΓÇô provides `UDPsocket`, `UDPpacket` types and UDP socket management
