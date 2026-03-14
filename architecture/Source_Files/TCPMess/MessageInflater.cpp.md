# Source_Files/TCPMess/MessageInflater.cpp

## File Purpose
Implements the MessageInflater class, which deserializes (inflates) network messages from wire format into Message objects. Acts as a registry-based factory that maintains prototype instances for each message type and clones them during deserialization.

## Core Responsibilities
- **Deserialize messages**: Inflate UninflatedMessage instances into typed Message objects via prototype cloning and population
- **Prototype management**: Register and unregister message type prototypes by MessageTypeID
- **Type-driven instantiation**: Use message type metadata to locate the correct prototype and instantiate appropriate Message subclass
- **Error resilience**: Fall back to uninflated message on deserialization failure (clone of source)
- **Memory management**: Maintain prototype instances and clean up on removal/destruction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MessageInflaterMap | typedef (std::map) | Maps MessageTypeID ΓåÆ Message* prototype instances |

## Global / File-Static State
None.

## Key Functions / Methods

### inflate
- **Signature:** `Message* MessageInflater::inflate(const UninflatedMessage& inSource)`
- **Purpose:** Deserialize a wire-format message into a typed Message object.
- **Inputs:** `inSource` ΓÇö UninflatedMessage with type metadata and serialized data
- **Outputs/Return:** Pointer to inflated Message (or clone of source if deserialization fails); never NULL
- **Side effects:** Allocates new Message object via prototype clone; may delete temporary clone on exception
- **Calls:** `inSource.inflatedType()`, `i->second->clone()`, `theResult->inflateFrom()`, `inSource.clone()`
- **Notes:** 
  - Catches all exceptions during clone/inflate; gracefully degrades to uninflated fallback
  - Caller responsible for deleting returned pointer
  - Returns a Message (original uninflated copy) if prototype not found or inflate fails

### learnPrototypeForType
- **Signature:** `void MessageInflater::learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- **Purpose:** Register a new message prototype for a type ID (or replace existing).
- **Inputs:** `inType` ΓÇö message type identifier; `inPrototype` ΓÇö Message subclass instance to use as template
- **Outputs/Return:** None
- **Side effects:** Clones inPrototype, deletes prior prototype if registered, updates mMap
- **Calls:** `inPrototype.clone()`
- **Notes:** Takes ownership of the cloned prototype; callers may safely delete their copy

### removePrototypeForType
- **Signature:** `void MessageInflater::removePrototypeForType(MessageTypeID inType)`
- **Purpose:** Unregister a message prototype by type ID.
- **Inputs:** `inType` ΓÇö message type identifier
- **Outputs/Return:** None
- **Side effects:** Deletes prototype, erases map entry
- **Calls:** None (STL map operations)
- **Notes:** Safe to call on non-existent type ID (no-op)

### ~MessageInflater
- **Signature:** `MessageInflater::~MessageInflater()`
- **Purpose:** Destructor; clean up all stored prototypes.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes all Message prototypes in mMap; clears map
- **Calls:** None (loop + delete)
- **Notes:** Ensures no memory leaks from prototype registry

## Control Flow Notes
Not inferable as a standalone component, but likely part of a network deserialization pipeline: on receiving a serialized message, code queries MessageInflater::inflate() to reconstruct the typed object. Registration of prototypes (learnPrototypeForType) occurs at initialization.

## External Dependencies
- `#include <map>` ΓÇö STL container for typeΓåÆprototype mapping
- `#include "Message.h"` ΓÇö defines Message, UninflatedMessage, MessageTypeID (not in this file)
- Conditional compilation: `#if !defined(DISABLE_NETWORKING)` ΓÇö entire file disabled if networking is off
