# Source_Files/Misc/sdl_network.h

## File Purpose
SDL-specific wrapper and data structure definitions for the network subsystem. Bridges the abstract network.h interface to SDL_net library, primarily handling UDP datagram operations (DDP) and TCP connections (ADSP/ConnectionEnd).

## Core Responsibilities
- Define SDL-native network data structures (DDPFrame, DDPPacketBuffer, ConnectionEnd)
- Specify addressing and packet buffer types (NetEntityName, NetAddrBlock)
- Declare function pointers for packet handlers and entity lookup callbacks
- Expose UDP socket lifecycle management (open, close, send)
- Provide type-safe wrappers for SDL_net socket objects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| DDPFrame | struct | UDP datagram frame with 1500-byte payload and associated UDPsocket handle |
| DDPPacketBuffer | struct | Received packet buffer with protocol type, source address, and datagram payload |
| ConnectionEnd | struct | TCP connection state holding socket, server socket, and socket set for async I/O |
| NetEntityName | typedef | 32-byte buffer for network entity names (player names, etc.) |
| NetAddrBlock | typedef | IP address wrapper (alias for SDL_net's IPaddress) |
| PacketHandlerProcPtr | function pointer | Callback signature for handling incoming UDP packets |
| lookupUpdateProcPtr | function pointer | Callback for entity lookup status (add/remove notifications) |
| lookupFilterProcPtr | function pointer | Callback to filter entities during lookup |

## Global / File-Static State
None.

## Key Functions / Methods

### NetDDPOpen / NetDDPClose
- Signature: `OSErr NetDDPOpen(void)` / `OSErr NetDDPClose(void)`
- Purpose: Initialize and shut down UDP datagram layer
- Inputs: None
- Outputs/Return: Error code (OSErr)
- Side effects: Allocates/deallocates global UDP socket state

### NetDDPOpenSocket / NetDDPCloseSocket
- Signature: `OSErr NetDDPOpenSocket(short *ioPortNumber, PacketHandlerProcPtr packetHandler)` / `OSErr NetDDPCloseSocket(short ignored)`
- Purpose: Open a UDP port with packet handler callback; close it (currently only one UDP socket supported)
- Inputs: Port number (in/out, network byte order), packet handler callback
- Outputs/Return: Error code; port number (out param)
- Side effects: Registers packet handler callback; binds UDP port
- Notes: Comments note port parameter is now used for IP port (was socket number in AppleTalk era). CloseSocket argument is ignored.

### NetDDPNewFrame / NetDDPDisposeFrame
- Signature: `DDPFramePtr NetDDPNewFrame(void)` / `void NetDDPDisposeFrame(DDPFramePtr frame)`
- Purpose: Allocate and free UDP frame structures
- Inputs: Frame pointer (dispose only)
- Outputs/Return: Frame pointer (new only)
- Side effects: Heap allocation/deallocation

### NetDDPSendFrame
- Signature: `OSErr NetDDPSendFrame(DDPFramePtr frame, NetAddrBlock *address, short protocolType, short socket)`
- Purpose: Send UDP datagram to remote address
- Inputs: Frame to send, destination address, protocol type, socket (deprecated)
- Outputs/Return: Error code
- Side effects: Network I/O
- Notes: Socket parameter usage unclear from signature alone

### NetState / NetSetServerIdentifier / NetEntityNotInGame
- Purpose: Query network state, set server identifier, filter entities not in-game
- Notes: Prototypes only; implementations elsewhere

## Control Flow Notes
This header defines the **low-level SDL network abstraction layer**. It sits between:
1. **Upper layer**: `network.h` (game state machine: gather, join, active, etc.)
2. **Lower layer**: SDL_net library (raw UDP/TCP sockets)

DDP (UDP) is the primary focus; ADSP (TCP) support is minimal (ConnectionEnd structure defined but prototypes removed per comments). Packet handlers are registered at socket open time to receive async incoming datagrams.

## External Dependencies
- **SDL_net.h**: UDP/TCP socket types (`UDPsocket`, `TCPsocket`, `SDLNet_SocketSet`)
- **cseries.h**: Core type definitions (`uint16`, `byte`, `OSErr`)
- **network.h**: High-level network interface and game state machine

## Notes
- Constant `ddpMaxData` (1500 bytes) is for **allocation only**, not actual packet size guarantee.
- Comments note SDL network microphone code sends "big" packets; UDP on Ethernet typically carries ~1500 bytes.
- ADSP/TCP comment mentions code now uses "TCPMess" instead of original ADSP protocol.
