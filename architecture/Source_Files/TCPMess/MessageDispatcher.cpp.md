# Source_Files/TCPMess/MessageDispatcher.cpp

## File Purpose
Minimal .cpp stub for the MessageDispatcher class that conditionally includes its header. The actual implementation is header-only, defined in `MessageDispatcher.h`. Provides a central message routing/dispatch mechanism for TCP-based networked communication.

## Core Responsibilities
- Route incoming messages to type-specific handlers based on message type ID
- Maintain a type-to-handler mapping and support handler registration/deregistration
- Fall back to a default handler when no type-specific handler exists
- Implement the MessageHandler interface to participate in the message handling chain

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MessageDispatcher | class | Central dispatcher; inherits MessageHandler and delegates to registered handlers |
| MessageDispatcherMap | typedef | `std::map<MessageTypeID, MessageHandler*>` ΓÇö maps message type IDs to handler pointers |

## Global / File-Static State
None. (Per-instance members: `mMap` and `mDefaultHandler` are defined in the header.)

## Key Functions / Methods

### setHandlerForType
- Signature: `void setHandlerForType(MessageHandler* inHandler, MessageTypeID inType)`
- Purpose: Register or unregister a handler for a specific message type.
- Inputs: `inHandler` (handler pointer, may be NULL), `inType` (message type ID)
- Outputs/Return: None
- Side effects: Modifies `mMap`; if `inHandler` is NULL, clears the entry via `clearHandlerForType`.
- Calls: `clearHandlerForType`
- Notes: Allows NULL to unregister; provides convenience over direct map manipulation.

### handlerForType
- Signature: `MessageHandler* handlerForType(MessageTypeID inType)`
- Purpose: Look up the handler for a message type, with default fallback.
- Inputs: `inType` (message type ID)
- Outputs/Return: Handler pointer; returns `mDefaultHandler` if no type-specific handler found.
- Side effects: None (read-only lookup)
- Calls: None
- Notes: Always returns a handler (possibly the default); never returns NULL if default is set.

### handlerForTypeNoDefault
- Signature: `MessageHandler* handlerForTypeNoDefault(MessageTypeID inType)`
- Purpose: Look up the handler for a message type without falling back to default.
- Inputs: `inType` (message type ID)
- Outputs/Return: Handler pointer or NULL if not found.
- Side effects: None (read-only lookup)
- Calls: None

### handle
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- Purpose: Dispatch an incoming message to its appropriate handler (virtual override of MessageHandler).
- Inputs: `inMessage` (message to dispatch), `inChannel` (associated communication channel)
- Outputs/Return: None
- Side effects: Delegates to the handler, which may perform I/O or modify game state.
- Calls: `handlerForType`, then `theHandler->handle()`
- Notes: Safe null-check on handler before delegation; silently drops messages with no handler.

**Trivial helpers summarized**: `clearHandlerForType` (erases from map), `setDefaultHandler` (stores pointer), `defaultHandler()` (returns stored pointer).

## Control Flow Notes
Part of a message dispatch pipeline. Sits between incoming message reception (CommunicationsChannel) and handler execution. On frame or message-receipt event, `handle()` is invoked; it routes to type-specific or default handlers, decoupling message delivery from handling logic.

## External Dependencies
- `#include <map>` ΓÇö standard library map container
- `#include "Message.h"` ΓÇö defines Message class and MessageTypeID
- `#include "MessageHandler.h"` ΓÇö defines MessageHandler base class
- `CommunicationsChannel*` used but not fully declared in this header (forward declaration or defined elsewhere)
