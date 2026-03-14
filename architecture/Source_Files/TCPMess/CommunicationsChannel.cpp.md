# Source_Files/TCPMess/CommunicationsChannel.cpp

## File Purpose

Implementation of bidirectional TCP message communication for a game engine. Manages non-blocking socket I/O, message queuing, serialization, and platform-specific socket handling (Windows/macOS/Linux). Provides both individual channel communication and a factory for accepting incoming connections.

## Core Responsibilities

- Establish and manage TCP socket connections with non-blocking I/O
- Buffer and queue incoming/outgoing messages with header-based framing
- Serialize message headers (magic + type + length) and deserialize incoming frames
- Pump socket data through state machines (receive header ΓåÆ receive body; send header ΓåÆ send body)
- Support synchronous receive with overall and inactivity timeouts
- Inflate received messages via registered `MessageInflater` and dispatch via registered `MessageHandler`
- Handle platform-specific socket setup (winsock2/OpenTransport/fcntl)
- Batch-flush outgoing messages across multiple channels
- Accept incoming connections via `CommunicationsChannelFactory`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CommunicationResult` | enum | Return status for low-level socket operations: `kIncomplete`, `kComplete`, `kError` |
| `MessageQueue` | typedef | `std::list<Message*>` for received/inflated messages |
| `UninflatedMessageQueue` | typedef | `std::list<UninflatedMessage*>` for outgoing messages awaiting transmission |
| `_TCPsocket` (macOS) | struct | OpenTransport socket wrapper (event codes, connection state, endpoints) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kMaximumMessageLength` | enum (int) | static | Reject incoming messages > 4 MB to prevent buffer exhaustion |
| `kSSRPumpInterval` | enum (int) | static | 50 ms delay between pump calls in synchronous receive loops |
| `kFlushPumpInterval` | enum (int) | static | 50 ms delay between pump calls during flush operations |
| `MakeTCPsocketNonBlocking` | function pointer | static | Forward declaration; platform-specific socket setup |
| `Our_TCP_Recv` (macOS) | function | static | macOS OpenTransport receive wrapper; custom async event handling |
| `AsyncTCPPopEvent` (macOS) | function | static | macOS OT event dequeue and dispatch; handles connect/disconnect/data |

## Key Functions / Methods

### Constructor: `CommunicationsChannel()`
- **Signature:** `CommunicationsChannel()`
- **Purpose:** Default constructor for outgoing connections.
- **Inputs:** None
- **Outputs/Return:** Instance with `mConnected = false`, null socket and message state.
- **Side effects:** Initializes `mTicksAtLastReceive` and `mTicksAtLastSend` via `SDL_GetTicks()`.
- **Calls:** SDL_GetTicks

### Constructor: `CommunicationsChannel(TCPsocket inSocket)`
- **Signature:** `CommunicationsChannel(TCPsocket inSocket)`
- **Purpose:** Constructor for incoming connections (called by factory).
- **Inputs:** `inSocket` ΓÇô existing TCP socket
- **Outputs/Return:** Instance with `mConnected = (inSocket != NULL)`, socket bound.
- **Side effects:** Sets `mTicksAtLastReceive` and `mTicksAtLastSend`.
- **Calls:** SDL_GetTicks

### `receive_some()`
- **Signature:** `CommunicationResult receive_some(TCPsocket inSocket, byte* inBuffer, size_t& ioBufferPosition, size_t inBufferLength)`
- **Purpose:** Low-level non-blocking receive; reads from socket into buffer at `ioBufferPosition`.
- **Inputs:** Socket, buffer, current position, target length
- **Outputs/Return:** `kIncomplete` (partial read), `kComplete` (filled), or `kError` (disconnect/bad errno).
- **Side effects:** Advances `ioBufferPosition` by bytes received; updates `mTicksAtLastReceive` on success; calls `disconnect()` on error.
- **Calls:** `SDLNet_TCP_Recv()` or `Our_TCP_Recv()` (macOS); platform-specific errno/WSAGetLastError checks.
- **Notes:** Handles `-1` returns ambiguously (sane vs. EAGAIN/WSAEWOULDBLOCK); macOS path defers to custom handler.

### `send_some()`
- **Signature:** `CommunicationResult send_some(TCPsocket inSocket, byte* inBuffer, size_t& ioBufferPosition, size_t inBufferLength)`
- **Purpose:** Low-level non-blocking send.
- **Inputs:** Socket, buffer, current position, target length
- **Outputs/Return:** `kIncomplete`, `kComplete`, or `kError`.
- **Side effects:** Advances `ioBufferPosition`; updates `mTicksAtLastSend` on success; disconnects on error.
- **Calls:** `SDLNet_TCP_Send()`

### `receiveHeader()`
- **Signature:** `bool receiveHeader()`
- **Purpose:** Receive 8-byte message header; deserialize magic, type, length; allocate `UninflatedMessage` if valid.
- **Inputs:** None (uses member `mSocket`, `mIncomingHeader`, `mIncomingHeaderPosition`)
- **Outputs/Return:** `true` if complete (loop again); `false` if incomplete.
- **Side effects:** On success, allocates new `UninflatedMessage` and resets header position; disconnects if magic/length invalid.
- **Calls:** `receive_some()`, `AIStreamBE` deserializer, `UninflatedMessage` constructor, `disconnect()`.
- **Notes:** Header includes 8-byte header in transmitted length; subtracts it before storing message length.

### `_receiveMessage()`
- **Signature:** `bool _receiveMessage()`
- **Purpose:** Receive message body; inflate if inflater registered; enqueue and reset state.
- **Inputs:** None (uses member `mSocket`, `mIncomingMessage`)
- **Outputs/Return:** `true` if complete; `false` otherwise.
- **Side effects:** Enqueues inflated or raw message; deletes `mIncomingMessage`; resets `mIncomingHeaderPosition`.
- **Calls:** `receive_some()`, `MessageInflater::inflate()`, `delete`.

### `sendHeader()`
- **Signature:** `bool sendHeader()`
- **Purpose:** Send 8-byte packed header.
- **Inputs:** None (uses member `mOutgoingHeader`, `mOutgoingHeaderPosition`)
- **Outputs/Return:** `true` if complete; `false` otherwise.
- **Side effects:** Resets `mOutgoingMessagePosition` on completion.
- **Calls:** `send_some()`

### `sendMessage()`
- **Signature:** `bool sendMessage()`
- **Purpose:** Send front message body; delete and dequeue on completion.
- **Inputs:** None (uses `mOutgoingMessages`, `mOutgoingMessagePosition`)
- **Outputs/Return:** `true` if complete; `false` otherwise.
- **Side effects:** Pops front message and deletes; resets `mOutgoingHeaderPosition` for next message.
- **Calls:** `send_some()`, `delete`

### `pump()`
- **Signature:** `void pump()`
- **Purpose:** Drive both send and receive state machines; main event loop entry point.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Advances internal I/O state; may enqueue/dequeue messages, update timestamps.
- **Calls:** `pumpSendingSide()`, `pumpReceivingSide()`

### `pumpReceivingSide()` / `pumpSendingSide()`
- **Purpose:** Loop state machine for receiving headers/bodies or sending headers/bodies until blocked.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** State transitions; message allocation/deallocation.
- **Calls:** `receiveHeader()`, `_receiveMessage()`, `sendHeader()`, `sendMessage()`
- **Notes:** `pumpSendingSide()` packs header lazily (once per message) via `AOStreamBE`.

### `receiveMessage()`
- **Signature:** `Message* receiveMessage(Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Blocking receive with two timeout layers: overall deadline and inactivity grace period.
- **Inputs:** Overall timeout (ms), inactivity timeout (ms)
- **Outputs/Return:** Pointer to received (inflated) message, or NULL if timeout/disconnect.
- **Side effects:** Pumps repeatedly via `SDL_Delay()`; caller owns returned message.
- **Calls:** `pump()`, `SDL_GetTicks()`, `SDL_Delay()`
- **Notes:** Inactivity timer resets on each data reception; overall timer is hard deadline.

### `receiveSpecificMessage()`
- **Signature:** `Message* receiveSpecificMessage(MessageTypeID inType, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Blocking receive for a specific message type; dispatch other messages via handler.
- **Inputs:** Desired type ID, timeouts
- **Outputs/Return:** Matching message pointer, or NULL.
- **Side effects:** Calls `receiveMessage()` in a loop; dispatches non-matching messages.
- **Calls:** `receiveMessage()`, `messageHandler()->handle()`, `delete`

### `enqueueOutgoingMessage()`
- **Signature:** `void enqueueOutgoingMessage(const Message& inMessage)`
- **Purpose:** Copy and queue outgoing message if connected.
- **Inputs:** Message reference (const)
- **Outputs/Return:** None
- **Side effects:** Calls `Message::deflate()` and enqueues result.
- **Calls:** `isConnected()`, `Message::deflate()`

### `flushOutgoingMessages()`
- **Signature:** `void flushOutgoingMessages(bool shouldDispatchIncomingMessages, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Blocking wait for all queued outgoing messages to be delivered; optionally dispatch incoming.
- **Inputs:** Flag to dispatch, timeouts
- **Outputs/Return:** None
- **Side effects:** Loops via `pump()` and optional `dispatchIncomingMessages()` until queue empty or timeout.
- **Calls:** `pump()`, `dispatchIncomingMessages()`, `SDL_Delay()`

### `multipleFlushOutgoingMessages()`
- **Signature:** `static void multipleFlushOutgoingMessages(std::vector<CommunicationsChannel*>&, bool, Uint32, Uint32)`
- **Purpose:** Batch flush across multiple channels; more efficient than individual flushes.
- **Inputs:** Channel vector, dispatch flag, timeouts
- **Outputs/Return:** None
- **Side effects:** Polls all channels until all queues empty or overall timeout.
- **Calls:** `pump()`, `dispatchIncomingMessages()`

### `dispatchIncomingMessages()` / `dispatchOneIncomingMessage()`
- **Signature:** `void dispatchIncomingMessages()` / `bool dispatchOneIncomingMessage()`
- **Purpose:** Invoke handler callback for each queued received message; delete message after handling.
- **Inputs:** None
- **Outputs/Return:** `void` / `bool` (true if dispatched at least one).
- **Side effects:** Calls `MessageHandler::handle()`, `delete`.
- **Calls:** `messageHandler()->handle()`

### `connect(const std::string&, uint16)` / `connect(const IPaddress&)`
- **Signature:** Two overloads; string resolves host name/port; IPaddress opens directly.
- **Purpose:** Establish outgoing connection; reset state; enable non-blocking mode.
- **Inputs:** Host string + port, or packed IPaddress (port in network byte order).
- **Outputs/Return:** None
- **Side effects:** Allocates socket via `SDLNet_TCP_Open()`; clears queues and buffers; calls `MakeTCPsocketNonBlocking()`.
- **Calls:** `SDLNet_ResolveHost()` (string path), `SDLNet_TCP_Open()`, `MakeTCPsocketNonBlocking()`, `SDL_GetTicks()`

### `disconnect()`
- **Signature:** `void disconnect()`
- **Purpose:** Close socket and discard all queued messages.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes socket, clears incoming/outgoing queues, resets positions.
- **Calls:** `SDLNet_TCP_Close()`, `delete` (queue cleanup)

### `CommunicationsChannelFactory::newIncomingConnection()`
- **Signature:** `CommunicationsChannel* newIncomingConnection()`
- **Purpose:** Poll server socket; accept and wrap new incoming connection if available.
- **Inputs:** None
- **Outputs/Return:** New `CommunicationsChannel*` with accepted socket, or NULL.
- **Side effects:** Allocates socket set, checks socket, accepts connection, makes non-blocking.
- **Calls:** `SDLNet_AllocSocketSet()`, `SDLNet_TCP_AddSocket()`, `SDLNet_CheckSockets()`, `SDLNet_TCP_Accept()`, `MakeTCPsocketNonBlocking()`, `SDLNet_FreeSocketSet()`

### `MakeTCPsocketNonBlocking()`
- **Signature:** `static void MakeTCPsocketNonBlocking(TCPsocket* socket)`
- **Purpose:** Platform-specific non-blocking socket setup (exploits SDL_net struct internals).
- **Inputs:** Pointer to socket
- **Outputs/Return:** None
- **Side effects:** Calls platform I/O control (ioctlsocket/OTSetNonBlocking/fcntl).
- **Calls:** `ioctlsocket()` (WIN32), `OTSetNonBlocking()` (macOS), `fcntl()` (Unix)
- **Notes:** Accesses socket FD via pointer cast; brittle if SDL_net struct changes.

### `Our_TCP_Recv()` (macOS only)
- **Signature:** `static int Our_TCP_Recv(_TCPsocket* sock, void* data, int maxlen)`
- **Purpose:** macOS OpenTransport receive wrapper with custom event handling.
- **Inputs:** Socket, buffer, max bytes
- **Outputs/Return:** Bytes received, or -1 on error.
- **Side effects:** Calls `OTRcv()`, `AsyncTCPPopEvent()`, `OTLook()`.
- **Calls:** Platform OT APIs

## Control Flow Notes

1. **Initialization:** Constructor ΓåÆ (optional) `connect()` ΓåÆ socket ready, non-blocking.
2. **Pump cycle (driven by user or sync-receive loop):**
   - `pump()` ΓåÆ `pumpSendingSide()` checks outgoing queue; builds header (if needed) ΓåÆ sends header ΓåÆ sends body
   - `pump()` ΓåÆ `pumpReceivingSide()` alternates: receive header (if no body in progress) or receive body
3. **Synchronous receive:** `receiveMessage()` loops `pump() + SDL_Delay()` until message or timeout.
4. **Dispatch:** User calls `dispatchIncomingMessages()` (often from timeout/flush loops) to invoke handlers.
5. **Shutdown:** `disconnect()` closes socket and drains queues.

## External Dependencies

- **SDL_net:** `SDLNet_TCP_Open()`, `SDLNet_TCP_Close()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`, `SDLNet_TCP_Accept()`, `SDLNet_TCP_GetPeerAddress()`, `SDLNet_ResolveHost()`, `SDLNet_AllocSocketSet()`, `SDLNet_FreeSocketSet()`, `SDLNet_CheckSockets()`, `SDLNet_TCP_AddSocket()`, `SDLNet_SetError()`
- **SDL:** `SDL_GetTicks()`, `SDL_Delay()`, `SDL_SwapBE16()`, `SDL_GetError()`
- **Platform APIs:** `winsock2.h` (ioctlsocket, WSAGetLastError), `OpenTransport.h` (macOS OT functions), `fcntl.h` (Unix fcntl)
- **Game engine:** `Message`, `UninflatedMessage`, `MessageInflater`, `MessageHandler` (defined elsewhere)
- **Serialization:** `AIStreamBE`, `AOStreamBE` (AStream.h)
- **Utilities:** `cseries.h` (SDL wrapper, type defs), `cstypes.h` (uint8, uint16, etc.), `errno.h`
