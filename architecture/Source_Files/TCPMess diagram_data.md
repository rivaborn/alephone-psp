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

## External Dependencies

- **SDL_net:** `SDLNet_TCP_Open()`, `SDLNet_TCP_Close()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`, `SDLNet_TCP_Accept()`, `SDLNet_TCP_GetPeerAddress()`, `SDLNet_ResolveHost()`, `SDLNet_AllocSocketSet()`, `SDLNet_FreeSocketSet()`, `SDLNet_CheckSockets()`, `SDLNet_TCP_AddSocket()`, `SDLNet_SetError()`
- **SDL:** `SDL_GetTicks()`, `SDL_Delay()`, `SDL_SwapBE16()`, `SDL_GetError()`
- **Platform APIs:** `winsock2.h` (ioctlsocket, WSAGetLastError), `OpenTransport.h` (macOS OT functions), `fcntl.h` (Unix fcntl)
- **Game engine:** `Message`, `UninflatedMessage`, `MessageInflater`, `MessageHandler` (defined elsewhere)
- **Serialization:** `AIStreamBE`, `AOStreamBE` (AStream.h)
- **Utilities:** `cseries.h` (SDL wrapper, type defs), `cstypes.h` (uint8, uint16, etc.), `errno.h`

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

## External Dependencies
- **SDL_net**: `TCPsocket`, `IPaddress`, `Uint8`, `Uint16`, `Uint32`, `SDL_GetTicks()`.
- **Message.h** (bundled): `Message` (base), `UninflatedMessage` (serialized form), `MessageInflater` (deserializer callback), `MessageHandler` (message dispatch callback), `MessageTypeID`.
- **Standard C++**: `<list>`, `<string>`, `<memory>`, `<stdexcept>`, `<vector>`, `std::auto_ptr`, `std::runtime_error`.

# Source_Files/TCPMess/Message.cpp
## File Purpose
Implements serialization and deserialization for network message types in a game engine. Provides two concrete message implementations: `SmallMessageHelper` (variable-size messages using streams) and `BigChunkOfDataMessage` (binary blob storage). Conditionally compiled only when `DISABLE_NETWORKING` is not defined.

## Core Responsibilities
- Serialize/deserialize small variable-sized messages via stream-based encoding (inflate/deflate pattern)
- Store and manage large binary data payloads with explicit type IDs
- Handle buffer allocation and memory management for message data
- Support message cloning for data duplication
- Enforce 4 KB size limit for small message serialization buffer

## External Dependencies
- **Message.h:** Abstract `Message` base class; `UninflatedMessage` container; type ID constants
- **AStream.h:** `AIStreamBE`, `AOStreamBE` (big-endian serialization streams); `AIStream` base for polymorphic extraction
- **SDL.h** (indirectly via Message.h): `Uint8`, `Uint16` types
- **<string.h>:** `memcpy()` for buffer copying
- **<vector>:** `std::vector<byte>` for temporary serialization buffer in `SmallMessageHelper::deflate()`
- **COVARIANT_RETURN macro:** Conditional covariant return type (MSVC version check in Message.h)

# Source_Files/TCPMess/Message.h
## File Purpose
Defines an abstract message protocol system for serializing and deserializing structured data, likely for TCP network communication. Provides base classes and templates for creating type-safe, inflation/deflation-based message objects that can be transmitted as raw bytes.

## Core Responsibilities
- Define abstract `Message` base class with virtual serialization interface
- Provide `UninflatedMessage` wrapper for raw message byte buffers with type metadata
- Offer concrete message implementations for different payload patterns (dataless, simple scalars, large binary chunks, stream-based)
- Manage message type identification via `MessageTypeID`
- Handle deep copy and ownership semantics for message data

## External Dependencies
- **SDL.h:** Uint8, Uint16 type definitions
- **string.h:** memcpy for buffer copying
- **AIStream, AOStream:** Forward-declared input/output stream types (defined elsewhere)

# Source_Files/TCPMess/MessageDispatcher.cpp
## File Purpose
Minimal .cpp stub for the MessageDispatcher class that conditionally includes its header. The actual implementation is header-only, defined in `MessageDispatcher.h`. Provides a central message routing/dispatch mechanism for TCP-based networked communication.

## Core Responsibilities
- Route incoming messages to type-specific handlers based on message type ID
- Maintain a type-to-handler mapping and support handler registration/deregistration
- Fall back to a default handler when no type-specific handler exists
- Implement the MessageHandler interface to participate in the message handling chain

## External Dependencies
- `#include <map>` ΓÇö standard library map container
- `#include "Message.h"` ΓÇö defines Message class and MessageTypeID
- `#include "MessageHandler.h"` ΓÇö defines MessageHandler base class
- `CommunicationsChannel*` used but not fully declared in this header (forward declaration or defined elsewhere)

# Source_Files/TCPMess/MessageDispatcher.h
## File Purpose
Routes incoming network messages to type-specific handlers based on message type ID. Implements a dispatcher pattern with fallback to a default handler for unregistered message types.

## Core Responsibilities
- Maintain a registry mapping message type IDs to handler instances
- Dispatch incoming messages to the appropriate handler via polymorphism
- Support a default handler for message types without specific handlers
- Provide type-safe handler lookup with and without fallback
- Allow dynamic registration/unregistration of handlers

## External Dependencies
- `<map>` ΓÇö std::map container
- `"Message.h"` ΓÇö Message base class, UninflatedMessage, MessageTypeID (Uint16)
- `"MessageHandler.h"` ΓÇö MessageHandler interface and concrete handler templates (TypedMessageHandlerFunction, MessageHandlerMethod)
- `CommunicationsChannel` ΓÇö opaque type passed through to handlers; likely represents a network connection or socket

# Source_Files/TCPMess/MessageHandler.cpp
## File Purpose
Implementation stub for the MessageHandler system. All meaningful code is defined inline in the header; this file contains only conditional compilation guards and the header include.

## Core Responsibilities
- Include MessageHandler.h declarations (conditionally compiled when networking is enabled)
- Serve as the implementation unit for the message handling callback system

## External Dependencies
- `#include <cstdlib>` (header)
- Forward declared: `Message`, `CommunicationsChannel` (defined elsewhere)
- Conditional: `DISABLE_NETWORKING` guard wraps entire file

# Source_Files/TCPMess/MessageHandler.h
## File Purpose
Defines the abstract interface and template-based implementations for message handlers in a TCP messaging system. Enables type-safe dispatch of incoming messages to registered handler functions or methods via a polymorphic handler framework.

## Core Responsibilities
- Define abstract `MessageHandler` base class for message dispatching
- Provide `TypedMessageHandlerFunction` template for wrapping free functions as message handlers
- Provide `MessageHandlerMethod` template for wrapping class methods as message handlers
- Support dynamic casting of messages and channels to their concrete types
- Enable flexible handler registration without explicit subclassing

## External Dependencies
- `<cstdlib>` ΓÇô standard library (included but unused; likely legacy)
- **Forward declarations** (defined elsewhere): `Message`, `CommunicationsChannel`
- No direct dependencies on other project files visible in this header

# Source_Files/TCPMess/MessageInflater.cpp
## File Purpose
Implements the MessageInflater class, which deserializes (inflates) network messages from wire format into Message objects. Acts as a registry-based factory that maintains prototype instances for each message type and clones them during deserialization.

## Core Responsibilities
- **Deserialize messages**: Inflate UninflatedMessage instances into typed Message objects via prototype cloning and population
- **Prototype management**: Register and unregister message type prototypes by MessageTypeID
- **Type-driven instantiation**: Use message type metadata to locate the correct prototype and instantiate appropriate Message subclass
- **Error resilience**: Fall back to uninflated message on deserialization failure (clone of source)
- **Memory management**: Maintain prototype instances and clean up on removal/destruction

## External Dependencies
- `#include <map>` ΓÇö STL container for typeΓåÆprototype mapping
- `#include "Message.h"` ΓÇö defines Message, UninflatedMessage, MessageTypeID (not in this file)
- Conditional compilation: `#if !defined(DISABLE_NETWORKING)` ΓÇö entire file disabled if networking is off

# Source_Files/TCPMess/MessageInflater.h
## File Purpose
Declares a factory/registry class that deserializes raw "uninflated" message buffers into fully-typed `Message` objects. Uses the prototype pattern: stores message type templates and clones them to construct instances of the correct derived type based on message ID.

## Core Responsibilities
- Register message prototypes by type ID (`learnPrototype`, `learnPrototypeForType`)
- Deserialize `UninflatedMessage` buffers into typed `Message` instances (`inflate`)
- Unregister prototypes when no longer needed (`removePrototypeForType`)
- Manage lifetime of prototype instances (destructor cleanup)

## External Dependencies
- **Includes:** `<map>` (std::map), `"Message.h"` (Message, UninflatedMessage, MessageTypeID)
- **Defined elsewhere:** Implementation of `inflate`, `learnPrototypeForType`, `removePrototypeForType`, `~MessageInflater` (presumed in .cpp)


