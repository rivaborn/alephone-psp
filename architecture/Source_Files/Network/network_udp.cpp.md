# Source_Files/Network/network_udp.cpp

## File Purpose
Implements UDP-based datagram networking for the Aleph One game engine, replacing the original AppleTalk DDP protocol. Provides socket management, packet reception via a dedicated background thread, and packet transmission to remote machines.

## Core Responsibilities
- Initialize/shutdown the UDP networking module
- Open and close UDP sockets on specified ports
- Allocate and deallocate frame buffers for outgoing packets
- Receive incoming UDP packets in a background thread
- Invoke registered packet handlers when data arrives
- Send UDP frames to remote addresses
- Manage socket sets and thread lifecycle with proper synchronization

## Key Types / Data Structures
| Name | Kind (struct/enum/class/typedef) | Purpose |
|------|----------------------------------|---------|
| `DDPFrame` | struct | Outgoing packet frame with payload and socket handle |
| `DDPPacketBuffer` | struct | Received packet with protocol type, source address, and datagram data |
| `UDPsocket` | typedef | SDL_net UDP socket handle |
| `SDLNet_SocketSet` | typedef | SDL_net socket set for polling operations |
| `PacketHandlerProcPtr` | typedef | Function pointer for packet reception callback |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sUDPPacketBuffer` | `UDPpacket*` | static | Storage for raw incoming packet data from SDL_net |
| `ddpPacketBuffer` | `DDPPacketBuffer` | static | Processed DDP packet passed to handler callback |
| `sSocket` | `UDPsocket` | static | Single active UDP socket handle |
| `sSocketSet` | `SDLNet_SocketSet` | static | Socket set used by receiving thread for polling |
| `sPacketHandler` | `PacketHandlerProcPtr` | static | Registered callback invoked when packets arrive |
| `sReceivingThread` | `SDL_Thread*` | static | Background thread handle for packet reception |
| `sKeepListening` | `volatile bool` | static | Flag to signal receiving thread shutdown |

## Key Functions / Methods

### receive_thread_function
- **Signature:** `static int receive_thread_function(void*)`
- **Purpose:** Background thread loop that polls the UDP socket and dispatches incoming packets to the registered handler.
- **Inputs:** None (ignores void* parameter)
- **Outputs/Return:** Always returns 0
- **Side effects:** Acquires/releases `mytm_mutex`; calls `sPacketHandler` with received data; calls `fdprintf` on error
- **Calls:** `SDLNet_CheckSockets`, `SDLNet_UDP_Recv`, `take_mytm_mutex`, `release_mytm_mutex`, `memcpy`, `sPacketHandler` (function pointer)
- **Notes:** Polls with 1-second timeout to allow responsive shutdown. Drops packets if mutex cannot be acquired. Thread-safe because packet handler executes within mutex-protected critical section.

### NetDDPOpen
- **Signature:** `OSErr NetDDPOpen(void)`
- **Purpose:** Module initialization (currently a no-op placeholder).
- **Inputs:** None
- **Outputs/Return:** Always returns 0 (noErr)
- **Side effects:** None
- **Calls:** None

### NetDDPClose
- **Signature:** `OSErr NetDDPClose(void)`
- **Purpose:** Module shutdown (currently a no-op placeholder).
- **Inputs:** None
- **Outputs/Return:** Always returns 0 (noErr)
- **Side effects:** None
- **Calls:** None

### NetDDPOpenSocket
- **Signature:** `OSErr NetDDPOpenSocket(short *ioPortNumber, PacketHandlerProcPtr packetHandler)`
- **Purpose:** Open a UDP socket on the specified port and start the packet reception background thread.
- **Inputs:** `ioPortNumber` (port in network byte order); `packetHandler` (callback function)
- **Outputs/Return:** 0 on success, -1 on failure; port is currently echoed back unchanged (TODO note indicates real port should be returned)
- **Side effects:** Allocates `sUDPPacketBuffer`, opens `sSocket`, creates `sSocketSet`, spawns `sReceivingThread`, boosts thread priority
- **Calls:** `SDLNet_AllocPacket`, `SDL_SwapBE16`, `SDLNet_UDP_Open`, `SDLNet_AllocSocketSet`, `SDLNet_UDP_AddSocket`, `SDL_CreateThread`, `BoostThreadPriority`, `fdprintf`
- **Notes:** Swaps port from network byte order (big-endian) to host byte order for SDL_net. Thread priority boost may fail (logged as warning but not fatal).

### NetDDPCloseSocket
- **Signature:** `OSErr NetDDPCloseSocket(short portNumber)`
- **Purpose:** Close the UDP socket and shut down the receiving thread.
- **Inputs:** `portNumber` (currently ignored)
- **Outputs/Return:** Always returns 0 (noErr)
- **Side effects:** Signals `sReceivingThread` to exit via `sKeepListening` flag; waits for thread completion; frees `sSocketSet` and `sUDPPacketBuffer`; closes `sSocket`
- **Calls:** `SDL_WaitThread`, `SDLNet_FreeSocketSet`, `SDLNet_FreePacket`, `SDLNet_UDP_Close`
- **Notes:** Proper shutdown sequence ensures thread exits before resources are freed.

### NetDDPNewFrame
- **Signature:** `DDPFramePtr NetDDPNewFrame(void)`
- **Purpose:** Allocate a new outgoing packet frame.
- **Inputs:** None
- **Outputs/Return:** Pointer to initialized `DDPFrame` (or NULL on allocation failure)
- **Side effects:** Allocates heap memory; zeros frame structure; copies current socket handle
- **Calls:** `malloc`, `memset`
- **Notes:** Frame socket is populated with `sSocket` at allocation time.

### NetDDPDisposeFrame
- **Signature:** `void NetDDPDisposeFrame(DDPFramePtr frame)`
- **Purpose:** Release an outgoing packet frame.
- **Inputs:** `frame` (pointer to frame, may be NULL)
- **Outputs/Return:** None
- **Side effects:** Frees heap memory
- **Calls:** `free`
- **Notes:** Safe to call with NULL pointer.

### NetDDPSendFrame
- **Signature:** `OSErr NetDDPSendFrame(DDPFramePtr frame, NetAddrBlock *address, short protocolType, short port)`
- **Purpose:** Send an outgoing frame to a remote address.
- **Inputs:** `frame` (outgoing packet); `address` (destination IP:port); `protocolType`, `port` (unused in this implementation)
- **Outputs/Return:** 0 on success, -1 on failure
- **Side effects:** Populates global `sUDPPacketBuffer` with frame data and destination address
- **Calls:** `memcpy`, `SDLNet_UDP_Send`
- **Notes:** Reuses global `sUDPPacketBuffer` for transmission; sets channel to -1 (broadcast). Does not validate frame size against `ddpMaxData` at runtime (asserts in debug only).

## Control Flow Notes
**Initialization:** `NetDDPOpen()` ΓåÆ `NetDDPOpenSocket()` allocates resources and spawns background thread.

**Runtime:** Incoming packets arrive asynchronously on `sReceivingThread`. The thread polls with 1-second timeout, unmarshals raw UDP data into `ddpPacketBuffer` (under mutex protection), and invokes `sPacketHandler` callback. Outgoing packets are sent synchronously via `NetDDPSendFrame()`.

**Shutdown:** `NetDDPCloseSocket()` sets `sKeepListening = false`, waits for thread to exit, then frees all resources. `NetDDPClose()` is currently unused.

## External Dependencies
**SDL_net:** `SDLNet_AllocPacket`, `SDLNet_FreePacket`, `SDLNet_UDP_Open`, `SDLNet_UDP_Close`, `SDLNet_AllocSocketSet`, `SDLNet_FreeSocketSet`, `SDLNet_UDP_AddSocket`, `SDLNet_CheckSockets`, `SDLNet_UDP_Recv`, `SDLNet_UDP_Send`, `SDL_SwapBE16`

**SDL:** `SDL_Thread`, `SDL_CreateThread`, `SDL_WaitThread`

**Game engine internals:** `mytm_mutex` (mutual exclusion), `BoostThreadPriority` (thread scheduling), `fdprintf` (debug logging), `PacketHandlerProcPtr` callback type

**C standard library:** `malloc`, `free`, `memcpy`, `memset`

**Defined elsewhere:** `ddpMaxData` (max packet size), `DDPFrame`, `DDPPacketBuffer`, `kPROTOCOL_TYPE` constants
