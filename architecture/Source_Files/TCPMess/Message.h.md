# Source_Files/TCPMess/Message.h

## File Purpose
Defines an abstract message protocol system for serializing and deserializing structured data, likely for TCP network communication. Provides base classes and templates for creating type-safe, inflation/deflation-based message objects that can be transmitted as raw bytes.

## Core Responsibilities
- Define abstract `Message` base class with virtual serialization interface
- Provide `UninflatedMessage` wrapper for raw message byte buffers with type metadata
- Offer concrete message implementations for different payload patterns (dataless, simple scalars, large binary chunks, stream-based)
- Manage message type identification via `MessageTypeID`
- Handle deep copy and ownership semantics for message data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageTypeID` | typedef | Uint16 identifier for message types |
| `Message` | abstract class | Base interface for all message types |
| `UninflatedMessage` | class | Concrete wrapper holding raw bytes, type ID, and length |
| `SmallMessageHelper` | abstract class | Base for messages using stream-based serialization |
| `BigChunkOfDataMessage` | class | Wrapper for large binary payloads with buffer management |
| `SimpleMessage<tValueType>` | template class | Generic container for single typed values |
| `DatalessMessage<tMessageType>` | template class | Empty message with only type information |

## Global / File-Static State
None.

## Key Functions / Methods

### Message (virtual interface)
- **Signature:** `virtual MessageTypeID type() const = 0`
- **Purpose:** Return the message's type identifier
- **Outputs/Return:** MessageTypeID enum value

### Message::inflateFrom
- **Signature:** `virtual bool inflateFrom(const UninflatedMessage& inUninflated) = 0`
- **Purpose:** Deserialize from raw bytes; may raise exception on failure
- **Inputs:** Reference to UninflatedMessage containing raw data
- **Outputs/Return:** bool (success/failure)

### Message::deflate
- **Signature:** `virtual UninflatedMessage* deflate() const = 0`
- **Purpose:** Serialize to raw bytes
- **Outputs/Return:** Newly allocated UninflatedMessage (caller must delete)

### Message::clone
- **Signature:** `virtual Message* clone() const = 0`
- **Purpose:** Create independent copy of message
- **Outputs/Return:** Newly allocated Message subclass instance (covariant return)

### UninflatedMessage::UninflatedMessage (constructor)
- **Signature:** `UninflatedMessage(MessageTypeID inType, size_t inLength, Uint8* inBytes = NULL)`
- **Purpose:** Construct raw message wrapper; assumes ownership of inBytes if provided, otherwise allocates
- **Inputs:** Type ID, buffer length, optional byte pointer
- **Side effects:** Allocates new buffer if inBytes is NULL

### UninflatedMessage::copyToThis
- **Signature:** `void copyToThis(const UninflatedMessage& inSource)`
- **Purpose:** Deep-copy source message data (used by copy constructor and assignment)
- **Side effects:** Allocates new buffer, copies via memcpy
- **Calls:** memcpy

### BigChunkOfDataMessage::copyBufferFrom
- **Signature:** `void copyBufferFrom(const Uint8* inBuffer, size_t inLength)`
- **Purpose:** Update internal buffer with new data (signature in class but implementation not shown)
- **Side effects:** Allocates/deallocates mBuffer

### SimpleMessage<tValueType>::reallyDeflateTo
- **Signature:** `void reallyDeflateTo(AOStream& inStream) const`
- **Purpose:** Write templated value to output stream
- **Inputs:** Reference to AOStream
- **Calls:** operator<< on AOStream

### SimpleMessage<tValueType>::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Read templated value from input stream
- **Inputs:** Reference to AIStream
- **Outputs/Return:** bool (success)
- **Calls:** operator>> on AIStream

## Control Flow Notes
This is a protocol definition file. Typical flow:
1. **Creation:** Instantiate appropriate Message subclass (SimpleMessage, DatalessMessage, etc.)
2. **Serialization:** Call `deflate()` to produce UninflatedMessage with raw bytes
3. **Transmission:** Send UninflatedMessage buffer over network
4. **Deserialization:** Reconstruct typed Message via `inflateFrom()` on received UninflatedMessage
5. **Usage:** Query `type()` and access payload via type-specific methods

## External Dependencies
- **SDL.h:** Uint8, Uint16 type definitions
- **string.h:** memcpy for buffer copying
- **AIStream, AOStream:** Forward-declared input/output stream types (defined elsewhere)
