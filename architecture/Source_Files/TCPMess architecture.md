# Subsystem Overview

## Purpose
Implements TCP-based bidirectional message communication for networked gameplay. Manages non-blocking socket I/O, message serialization/deserialization, type-based routing, and connection lifecycle across multiple endpoints.

## Key Files
| File | Role |
|------|------|
| CommunicationsChannel.h/cpp | TCP socket lifecycle, non-blocking I/O state machine, message queuing, synchronous/asynchronous receive |
| Message.h/cpp | Abstract message protocol; concrete implementations for variable-size (SmallMessageHelper) and binary (BigChunkOfDataMessage) payloads |
| MessageInflater.h/cpp | Prototype-based factory for deserializing UninflatedMessage buffers into typed Message instances by message type ID |
| MessageDispatcher.h/cpp | Type-to-handler registry; routes incoming messages to registered handlers with fallback default handler |
| MessageHandler.h/cpp | Abstract MessageHandler base class and template implementations (TypedMessageHandlerFunction, MessageHandlerMethod) for polymorphic dispatch |

## Core Responsibilities
- Establish and manage TCP socket connections with non-blocking I/O and platform-specific setup (winsock2/OpenTransport/fcntl)
- Buffer incoming/outgoing messages with header-based framing (magic + type + length) and serialize/deserialize via AIStreamBE/AOStreamBE
- Pump socket data through receive state machines (header ΓåÆ body) and send state machines (header ΓåÆ body)
- Register and dispatch incoming messages to type-specific handlers via MessageDispatcher with fallback to default handler
- Inflate (deserialize) received UninflatedMessage buffers into typed Message objects using registered MessageInflater prototypes
- Support synchronous receive with overall and inactivity timeouts via SDL_GetTicks()
- Batch-flush outgoing messages across multiple channels and track send/receive activity timestamps
- Accept incoming connections via CommunicationsChannelFactory using SDL_net socket acceptance
- Manage per-channel application state via Memento pattern

## Key Interfaces & Data Flow
**Exposes to other subsystems:**
- `CommunicationsChannel` ΓÇö manages individual TCP endpoint (connect, send, receive, connectivity state)
- `CommunicationsChannelFactory` ΓÇö accepts incoming TCP connections
- `MessageDispatcher` ΓÇö central router implementing MessageHandler interface for type-based message dispatch
- `MessageHandler`, `TypedMessageHandlerFunction`, `MessageHandlerMethod` ΓÇö handler registration/callback framework for message reception
- `MessageInflater` ΓÇö prototype factory for message deserialization by type ID

**Consumes from other subsystems:**
- SDL_net (`SDLNet_TCP_*`, `SDLNet_CheckSockets`, `SDLNet_ResolveHost`)
- SDL (`SDL_GetTicks`, `SDL_Delay`, `SDL_SwapBE16`)
- AStream (`AIStreamBE`, `AOStreamBE`) for big-endian serialization
- Game engine Message definitions (Message, UninflatedMessage, MessageTypeID)

## Runtime Role
Conditionally compiled when `DISABLE_NETWORKING` is not defined. During networked gameplay, CommunicationsChannels pump I/O on each frame via `SDLNet_CheckSockets()`, deserialize incoming UninflatedMessage instances through the registered MessageInflater, dispatch to registered handlers via MessageDispatcher, and batch-flush outgoing message queues. Inactivity detection uses timestamp tracking for timeout semantics.

## Notable Implementation Details
- Non-blocking I/O via state machines: receive transitions headerΓåÆbody; send transitions headerΓåÆbody
- Platform-specific socket setup: Windows ioctlsocket (FIONBIO), macOS OpenTransport functions, Unix fcntl (O_NONBLOCK)
- 4 KB serialization buffer size limit for SmallMessageHelper variable-size messages
- Prototype pattern for message instantiation: MessageInflater stores registered prototypes and clones them during deserialization
- Header framing: magic number + message type ID (Uint16) + payload length precedes each message body
