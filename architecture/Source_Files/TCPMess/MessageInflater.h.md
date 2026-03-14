# Source_Files/TCPMess/MessageInflater.h

## File Purpose
Declares a factory/registry class that deserializes raw "uninflated" message buffers into fully-typed `Message` objects. Uses the prototype pattern: stores message type templates and clones them to construct instances of the correct derived type based on message ID.

## Core Responsibilities
- Register message prototypes by type ID (`learnPrototype`, `learnPrototypeForType`)
- Deserialize `UninflatedMessage` buffers into typed `Message` instances (`inflate`)
- Unregister prototypes when no longer needed (`removePrototypeForType`)
- Manage lifetime of prototype instances (destructor cleanup)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageInflaterMap` | typedef (std::map) | Maps `MessageTypeID` ΓåÆ prototype `Message*` for registry lookup |

## Global / File-Static State
None.

## Key Functions / Methods

### inflate
- **Signature:** `Message* inflate(const UninflatedMessage& inSource)`
- **Purpose:** Deserialize an uninflated message buffer into a typed Message instance by looking up the matching prototype and cloning it.
- **Inputs:** Uninflated message (contains raw type ID and byte buffer)
- **Outputs/Return:** Dynamically allocated cloned Message instance (caller must `delete`)
- **Side effects (global state, I/O, alloc):** Heap allocation of new Message clone
- **Calls (direct calls visible in this file):** Not visible in header; likely calls `Message::clone()`
- **Notes:** Caller owns the returned pointer. Unknown: what happens if type ID not in map (likely nullptr or exception).

### learnPrototype
- **Signature:** `void learnPrototype(const Message& inPrototype)`
- **Purpose:** Register a message prototype; convenience wrapper that extracts type ID from the prototype.
- **Inputs:** A Message instance (used to read its type ID)
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Inserts/updates entry in `mMap`; may store pointer to input Message
- **Calls (direct calls visible in this file):** Calls `learnPrototypeForType(inPrototype.type(), inPrototype)`
- **Notes:** Inline implementation

### learnPrototypeForType
- **Signature:** `void learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- **Purpose:** Register a message prototype for a specific type ID (not inferred from the Message itself).
- **Inputs:** Type ID and Message prototype
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Inserts/updates entry in `mMap`; stores pointer to prototype
- **Calls (direct calls visible in this file):** None visible
- **Notes:** Implementation likely in .cpp file

### removePrototypeForType
- **Signature:** `void removePrototypeForType(MessageTypeID inType)`
- **Purpose:** Unregister and potentially deallocate a prototype.
- **Inputs:** Type ID to remove
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Removes entry from `mMap`; unknown if deletes the Message pointer
- **Calls (direct calls visible in this file):** None visible
- **Notes:** Implementation in .cpp; unclear whether caller retains ownership of unregistered prototypes

### ~MessageInflater
- **Signature:** `~MessageInflater()`
- **Purpose:** Clean up registered prototypes.
- **Side effects (global state, I/O, alloc):** Deallocates all `Message*` pointers stored in `mMap`
- **Notes:** Implicitly required to prevent leaks of cloned prototypes.

## Control Flow Notes
This is a factory/registry component, likely invoked during message deserialization in a networking or IPC pipeline. Not directly tied to render/update cycles; fits into incoming message processing (e.g., TCP receive ΓåÆ uninflate ΓåÆ dispatch).

## External Dependencies
- **Includes:** `<map>` (std::map), `"Message.h"` (Message, UninflatedMessage, MessageTypeID)
- **Defined elsewhere:** Implementation of `inflate`, `learnPrototypeForType`, `removePrototypeForType`, `~MessageInflater` (presumed in .cpp)
