# Source_Files/TCPMess/CommunicationsChannel.h

## File Purpose
Defines a TCP-based bidirectional messaging channel with support for asynchronous and synchronous message handling. Provides both endpoint management (for outgoing connections) and server-side acceptance (via `CommunicationsChannelFactory`) of incoming connections.

## Core Responsibilities
- Manage TCP socket lifecycle (connect, disconnect, connectivity state)
- Queue and transmit outgoing messages with flush/timeout semantics
- Receive, parse, and dispatch incoming messages with pluggable handlers
- Support both synchronous (blocking) and asynchronous (callback-driven) receive patterns
- Manage message serialization via configurable `MessageInflater` (deserializer)
- Track send/receive activity timestamps for inactivity detection
- Handle per-channel application state via `Memento` pattern

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Memento` | class | Base class for storing application-defined per-channel state (safe downcasting via `dynamic_cast`) |
| `CommunicationsChannel` | class | Main bidirectional TCP messaging endpoint |
| `CommunicationsChannelFactory` | class | Server-side listener that accepts incoming connections and creates new channels |
| `CommunicationResult` | enum | Internal: tracks outcome of low-level TCP operations (kIncomplete, kComplete, kError) |

## Global / File-Static State
None. Class-level enum constants (`kSSRAnyDataTimeout`, `kHeaderMagic`, etc.) are scoped, not global.

## Key Functions / Methods

### Constructor: `CommunicationsChannel()`
- Signature: `CommunicationsChannel(); CommunicationsChannel(TCPsocket inSocket);`
- Purpose: Create a new channel, optionally wrapping an existing socket.
- Inputs: Existing `TCPsocket` (optional).
- Outputs: Initialized channel object.
- Side effects: Initializes internal message queues, header buffers, and state tracking.
- Notes: Default constructor for outgoing connections; socket-based constructor for incoming connections from factory.

### `pump()`
- Signature: `void pump();`
- Purpose: Non-blocking I/O pump that processes incoming and outgoing data without invoking message handlers.
- Inputs: None.
- Outputs: None.
- Side effects: Moves bytes from TCP socket into `mIncomingMessages` queue and from `mOutgoingMessages` queue to socket; updates `mTicksAtLastReceive`/`mTicksAtLastSend`.
- Calls: `pumpReceivingSide()`, `pumpSendingSide()`.
- Notes: Should be called frequently (main loop); does not trigger message handler callbacks.

### `dispatchIncomingMessages()`
- Signature: `void dispatchIncomingMessages();`
- Purpose: Process all queued incoming messages via the registered `MessageHandler`.
- Inputs: None.
- Outputs: None.
- Side effects: Invokes `mMessageHandler->handle()` for each message in `mIncomingMessages`; clears queue.
- Calls: `mMessageHandler::handle()` (registered callback).
- Notes: Requires prior `pump()` calls to populate queue.

### `receiveMessage()`
- Signature: `Message* receiveMessage(Uint32 inOverallTimeout = kSSRAnyMessageTimeout, Uint32 inInactivityTimeout = kSSRAnyDataTimeout);`
- Purpose: Synchronous (blocking) receiveΓÇöwaits for any message to be fully received.
- Inputs: Overall timeout (ms), inactivity timeout (ms).
- Outputs: Pointer to newly allocated inflated `Message` object (or NULL on timeout/disconnect).
- Side effects: Blocks thread; updates `mTicksAtLastReceive`; dispatches non-matching messages to handler if one is registered.
- Calls: `pump()`, `receiveHeader()`, `_receiveMessage()`.
- Notes: Caller owns returned object; NULL on timeout or disconnect.

### `receiveSpecificMessage<tMessage>()` (templated)
- Signature: `tMessage* receiveSpecificMessage(MessageTypeID inType, Uint32 inOverallTimeout, Uint32 inInactivityTimeout);` and `tMessage* receiveSpecificMessage(Uint32 inOverallTimeout, Uint32 inInactivityTimeout);` (type inferred from template arg).
- Purpose: Synchronous receive for a specific message type; other messages are dispatched to handler.
- Inputs: Message type ID; timeouts.
- Outputs: Typed pointer to matching message (downcasted via `dynamic_cast<>`), or NULL.
- Side effects: Blocks; dispatches non-matching messages.
- Calls: `receiveMessage()` + `dynamic_cast<>()`.
- Notes: Type-safe variant using template; includes throw variant `receiveSpecificMessageOrThrow()`.

### `enqueueOutgoingMessage()`
- Signature: `void enqueueOutgoingMessage(const Message& inMessage);`
- Purpose: Queue a message for transmission; copies message bytes to avoid caller error.
- Inputs: Reference to `Message` object.
- Outputs: None.
- Side effects: Copies message; adds to `mOutgoingMessages` queue.
- Calls: `Message::deflate()`.
- Notes: Non-blocking; actual transmission requires `flushOutgoingMessages()` or `pump()`.

### `flushOutgoingMessages()`
- Signature: `void flushOutgoingMessages(bool dispatchIncomingMessages, Uint32 inOverallTimeout = kOutgoingOverallTimeout, Uint32 inInactivityTimeout = kOutgoingInactivityTimeout);`
- Purpose: Wait for all queued outgoing messages to be sent over TCP.
- Inputs: Whether to also dispatch incoming messages during flush; timeouts.
- Outputs: None.
- Side effects: Blocks until all `mOutgoingMessages` sent or timeout/disconnect; optionally calls `dispatchIncomingMessages()`.
- Calls: `pump()`, `dispatchIncomingMessages()` (if requested).
- Notes: "Overall" timeout caps total wait; "Inactivity" timeout detects stalled sends.

### `multipleFlushOutgoingMessages()` (static)
- Signature: `static void multipleFlushOutgoingMessages(std::vector<CommunicationsChannel*>&, bool dispatchIncomingMessages, Uint32 inOverallTimeout, Uint32 inInactivityTimeout);`
- Purpose: Flush outgoing messages on multiple channels in a single call (more efficient than calling flush per channel).
- Inputs: Vector of channel pointers; dispatch flag; timeouts.
- Outputs: None.
- Side effects: Blocks; flushes all channels in parallel.
- Calls: `pump()` on each channel.
- Notes: Common pattern in server scenarios with many concurrent connections.

### `connect()`
- Signature: `void connect(const std::string& inAddressString, Uint16 inPort); void connect(const IPaddress& inAddress);`
- Purpose: Establish outgoing TCP connection.
- Inputs: Address string + port (host byte order), or `IPaddress` struct (network byte order).
- Outputs: None.
- Side effects: Creates and binds TCP socket; sets `mConnected` flag.
- Calls: SDL_net functions.
- Notes: Throws exception on failure (via `std::runtime_error`).

### `disconnect()`
- Signature: `void disconnect();`
- Purpose: Close the TCP connection.
- Inputs: None.
- Outputs: None.
- Side effects: Closes socket; sets `mConnected = false`; clears queues.
- Notes: Safe to call when already disconnected.

## Control Flow Notes
- **Pump-and-dispatch (async)**: Main loop calls `pump()` repeatedly ΓåÆ bytes flow into `mIncomingMessages` queue ΓåÆ periodically calls `dispatchIncomingMessages()` ΓåÆ `MessageHandler` callback fires.
- **Synchronous blocking**: Application calls `receiveMessage()` or `receiveSpecificMessage()` ΓåÆ function blocks, internally calling `pump()` in loops until timeout or message arrives ΓåÆ returns inflated message.
- **Outgoing**: Application calls `enqueueOutgoingMessage()` (queues) ΓåÆ later calls `flushOutgoingMessages()` or relies on `pump()` to drain queue.
- **Server acceptance**: `CommunicationsChannelFactory` listens on a port; `newIncomingConnection()` returns a new `CommunicationsChannel` wrapping the accepted socket.

## External Dependencies
- **SDL_net**: `TCPsocket`, `IPaddress`, `Uint8`, `Uint16`, `Uint32`, `SDL_GetTicks()`.
- **Message.h** (bundled): `Message` (base), `UninflatedMessage` (serialized form), `MessageInflater` (deserializer callback), `MessageHandler` (message dispatch callback), `MessageTypeID`.
- **Standard C++**: `<list>`, `<string>`, `<memory>`, `<stdexcept>`, `<vector>`, `std::auto_ptr`, `std::runtime_error`.
