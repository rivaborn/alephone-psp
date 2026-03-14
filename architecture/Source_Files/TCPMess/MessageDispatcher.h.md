# Source_Files/TCPMess/MessageDispatcher.h

## File Purpose
Routes incoming network messages to type-specific handlers based on message type ID. Implements a dispatcher pattern with fallback to a default handler for unregistered message types.

## Core Responsibilities
- Maintain a registry mapping message type IDs to handler instances
- Dispatch incoming messages to the appropriate handler via polymorphism
- Support a default handler for message types without specific handlers
- Provide type-safe handler lookup with and without fallback
- Allow dynamic registration/unregistration of handlers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageDispatcherMap` | typedef | `std::map<MessageTypeID, MessageHandler*>` ΓÇö maps message type IDs to their corresponding handler pointers |

## Global / File-Static State
None.

## Key Functions / Methods

### MessageDispatcher (constructor)
- Signature: `MessageDispatcher()`
- Purpose: Initialize dispatcher with empty handler map and null default handler
- Inputs: None
- Outputs/Return: Instance
- Side effects: None
- Calls: (implicit std::map constructor)
- Notes: Default handler is initially NULL; map starts empty

### setHandlerForType
- Signature: `void setHandlerForType(MessageHandler* inHandler, MessageTypeID inType)`
- Purpose: Register or unregister a handler for a specific message type
- Inputs: Handler pointer (or NULL to unregister), message type ID
- Outputs/Return: None
- Side effects: Modifies `mMap` ΓÇö inserts, updates, or erases map entry
- Calls: `clearHandlerForType()` if handler is NULL
- Notes: Passing NULL removes the handler entry; no validation of handler lifetime

### handlerForType
- Signature: `MessageHandler* handlerForType(MessageTypeID inType)`
- Purpose: Retrieve the handler for a message type, with fallback to default handler
- Inputs: Message type ID
- Outputs/Return: Handler pointer (or default handler if type not in map)
- Side effects: None
- Calls: `std::map::find()`
- Notes: Returns `mDefaultHandler` if type ID not found

### handlerForTypeNoDefault
- Signature: `MessageHandler* handlerForTypeNoDefault(MessageTypeID inType)`
- Purpose: Retrieve handler for a type without default fallback
- Inputs: Message type ID
- Outputs/Return: Handler pointer or NULL if not found
- Side effects: None
- Calls: `std::map::find()`
- Notes: No fallback; allows detection of unregistered types

### handle (override)
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- Purpose: Route an incoming message to its registered handler
- Inputs: Message pointer, communications channel pointer
- Outputs/Return: None
- Side effects: Delegates message handling to target handler; target handler may perform I/O or state modification
- Calls: `handlerForType()` ΓåÆ target handler's `handle()` method
- Notes: Silently drops messages with NULL handlers; no error reporting

### Accessors
- `clearHandlerForType()`, `setDefaultHandler()`, `defaultHandler()` ΓÇö manage default handler and type-specific removal

## Control Flow Notes
This component sits in the network receive path: after a raw message is parsed into a `Message` object and deserialized, the dispatcher routes it to the appropriate handler. Likely called from a network I/O layer or message pump, not from the main game loop.

## External Dependencies
- `<map>` ΓÇö std::map container
- `"Message.h"` ΓÇö Message base class, UninflatedMessage, MessageTypeID (Uint16)
- `"MessageHandler.h"` ΓÇö MessageHandler interface and concrete handler templates (TypedMessageHandlerFunction, MessageHandlerMethod)
- `CommunicationsChannel` ΓÇö opaque type passed through to handlers; likely represents a network connection or socket
