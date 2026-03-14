# Source_Files/TCPMess/Message.cpp

## File Purpose
Implements serialization and deserialization for network message types in a game engine. Provides two concrete message implementations: `SmallMessageHelper` (variable-size messages using streams) and `BigChunkOfDataMessage` (binary blob storage). Conditionally compiled only when `DISABLE_NETWORKING` is not defined.

## Core Responsibilities
- Serialize/deserialize small variable-sized messages via stream-based encoding (inflate/deflate pattern)
- Store and manage large binary data payloads with explicit type IDs
- Handle buffer allocation and memory management for message data
- Support message cloning for data duplication
- Enforce 4 KB size limit for small message serialization buffer

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `UninflatedMessage` | Class | Raw wire-format message container (type ID + buffer); base format for serialization/deserialization |
| `SmallMessageHelper` | Class | Abstract base for compact messages using big-endian stream encoding |
| `BigChunkOfDataMessage` | Class | Concrete message storing opaque binary data with manual buffer lifecycle |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kSmallMessageBufferSize` | enum constant | static | 4 KB working buffer for small message serialization |

## Key Functions / Methods

### SmallMessageHelper::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Deserialize a message from raw wire format using big-endian stream parsing.
- **Inputs:** `inUninflated` ΓÇô raw message buffer and length from network
- **Outputs/Return:** `bool` ΓÇô success/failure (subclass `reallyInflateFrom()` may throw)
- **Side effects:** Modifies internal state via `reallyInflateFrom()` (implemented by subclass)
- **Calls:** `AIStreamBE` constructor, `reallyInflateFrom()` (abstract)
- **Notes:** Wraps buffer in big-endian input stream; exact deserialization logic deferred to subclass.

### SmallMessageHelper::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Serialize message to raw wire format and return heap-allocated result.
- **Inputs:** None (uses internal state)
- **Outputs/Return:** New `UninflatedMessage*` (caller owns); must be `delete`d
- **Side effects:** Allocates 4 KB temporary buffer, then heap-allocates result message
- **Calls:** `reallyDeflateTo()` (abstract), `AOStreamBE` constructor, `memcpy()`
- **Notes:** Uses fixed 4 KB buffer; assumes serialized data fits. No bounds checking visible.

### BigChunkOfDataMessage::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Import binary data from wire format (no parsing; direct copy).
- **Inputs:** `inUninflated` ΓÇô raw message with buffer
- **Outputs/Return:** `true` (always succeeds)
- **Side effects:** Replaces internal buffer via `copyBufferFrom()`
- **Calls:** `copyBufferFrom()`
- **Notes:** No validation; assumes caller has verified message integrity.

### BigChunkOfDataMessage::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Export binary data to wire format as-is (no transformation).
- **Inputs:** None
- **Outputs/Return:** New `UninflatedMessage*` with copy of internal buffer; caller owns
- **Side effects:** Heap allocates new message and copies buffer
- **Calls:** `UninflatedMessage` constructor, `memcpy()`
- **Notes:** Direct copy; no encoding or compression.

### BigChunkOfDataMessage::copyBufferFrom
- **Signature:** `void copyBufferFrom(const byte* inBuffer, size_t inLength)`
- **Purpose:** Replace internal buffer with a deep copy of new data.
- **Inputs:** `inBuffer` ΓÇô source pointer; `inLength` ΓÇô byte count
- **Outputs/Return:** None
- **Side effects:** Deletes old buffer, allocates new heap buffer, copies data
- **Calls:** `memcpy()`
- **Notes:** Handles zero-length case (sets `mBuffer = NULL`). Used in assignment operators and constructors.

### BigChunkOfDataMessage::clone
- **Signature:** `COVARIANT_RETURN(Message*, BigChunkOfDataMessage*) clone() const`
- **Purpose:** Create independent copy of this message.
- **Inputs:** None
- **Outputs/Return:** New `BigChunkOfDataMessage*` (or `Message*` on older MSVC); caller owns
- **Side effects:** Allocates new object and internal buffer via `copyBufferFrom()`
- **Calls:** Constructor (copy variant), `copyBufferFrom()`
- **Notes:** Covariant return type for pre-.NET MSVC compatibility.

### BigChunkOfDataMessage::~BigChunkOfDataMessage
- **Signature:** Destructor
- **Purpose:** Release allocated buffer.
- **Side effects:** Deallocates `mBuffer`
- **Calls:** `delete[]`

## Control Flow Notes
This file is part of a **serialization/deserialization pipeline** for network messages:
1. **Inflate phase:** Raw `UninflatedMessage` (wire format) ΓåÆ internal representation (game state)
2. **Deflate phase:** Internal representation ΓåÆ `UninflatedMessage` for transmission
3. Not directly involved in frame/render loops; called during network I/O handling.

## External Dependencies
- **Message.h:** Abstract `Message` base class; `UninflatedMessage` container; type ID constants
- **AStream.h:** `AIStreamBE`, `AOStreamBE` (big-endian serialization streams); `AIStream` base for polymorphic extraction
- **SDL.h** (indirectly via Message.h): `Uint8`, `Uint16` types
- **<string.h>:** `memcpy()` for buffer copying
- **<vector>:** `std::vector<byte>` for temporary serialization buffer in `SmallMessageHelper::deflate()`
- **COVARIANT_RETURN macro:** Conditional covariant return type (MSVC version check in Message.h)
