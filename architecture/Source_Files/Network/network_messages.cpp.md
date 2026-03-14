# Source_Files/Network/network_messages.cpp

## File Purpose
Implements serialization and deserialization (inflate/deflate) for all network message types used during game setup and multiplayer communication. Provides helper utilities for string packing and binary encoding, plus zlib-based compression for large data chunks (maps, physics, Lua scripts).

## Core Responsibilities
- Serialize/deserialize network message objects to/from binary streams
- Convert between C strings, Pascal strings, and stream format
- Handle endianness via `AIStreamBE`/`AOStreamBE` (big-endian) stream classes
- Compress/decompress large data payloads (maps, physics, Lua) using zlib
- Maintain network protocol compatibility by preserving struct field order and sizes
- Support message types: hello, joiner info, capabilities, topology, chat, colors, warnings, join acceptance, client info

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPlayer | struct (via network_private.h) | Network-serialized player information (addresses, IDs, player data) |
| NetTopology | struct (via network_private.h) | Game topology: players, game settings, random seed, difficulty, etc. |
| UninflatedMessage | class (via Message.h) | Wrapper for compressed/uncompressed binary message buffer |
| AIStream / AOStream | classes (via AStream.h) | Input/output stream abstractions for deserialize/serialize with bounds checking |

## Global / File-Static State
None.

## Key Functions / Methods

### write_string
- **Signature:** `static void write_string(AOStream& outputStream, const char *s)`
- **Purpose:** Write a null-terminated C string to an output stream.
- **Inputs:** Output stream reference, pointer to C string.
- **Outputs/Return:** None.
- **Side effects:** Writes `strlen(s) + 1` bytes (including null terminator) to stream.
- **Calls:** `strlen()`, `outputStream.write()`.
- **Notes:** Simple wrapper; no validation on string length.

### write_pstring
- **Signature:** `static void write_pstring(AOStream& outputStream, const unsigned char *s)`
- **Purpose:** Convert and write a Pascal string (length-prefixed) as a C string to stream.
- **Inputs:** Output stream reference, pointer to Pascal string.
- **Outputs/Return:** None.
- **Side effects:** Converts PascalΓåÆC format, writes to stream.
- **Calls:** `pstrcpy()`, `a1_p2cstr()`, `write_string()`.
- **Notes:** Uses stack buffer of 256 bytes; depends on Pascal string utility functions.

### read_string / read_pstring
- **Signature:** `static void read_string(AIStream& inputStream, char *s, size_t length)` / `static void read_pstring(AIStream& inputStream, unsigned char *s, size_t length)`
- **Purpose:** Deserialize null-terminated or Pascal strings from input stream with bounds checking.
- **Inputs:** Input stream, buffer pointer, max buffer length.
- **Outputs/Return:** Populates caller's buffer.
- **Side effects:** Reads from stream; truncates if buffer too small.
- **Calls:** `inputStream >> (int8&) c`, utility functions.
- **Notes:** Null-terminates result; read_pstring internally converts CΓåÆPascal.

### deflateNetPlayer / inflateNetPlayer
- **Signature:** `static void deflateNetPlayer(AOStream& outputStream, const NetPlayer &player)` / `static void inflateNetPlayer(AIStream& inputStream, NetPlayer &player)`
- **Purpose:** Serialize/deserialize a `NetPlayer` struct (addresses, identifier, stream_id, player data).
- **Inputs:** Stream reference, player struct.
- **Outputs/Return:** Populates struct or stream.
- **Side effects:** Reads/writes fixed binary format: 4+2 bytes (DSP address), 4+2 bytes (DDP address), identifiers, player_data.
- **Calls:** Stream operators (`>>`, `<<`), `write_pstring()`/`read_pstring()`.
- **Notes:** Comment warns these must be replaced if struct changes for forward compatibility; field order is critical.

### BigChunkOfZippedDataMessage::inflateFrom
- **Signature:** `bool BigChunkOfZippedDataMessage::inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Decompress zlib-compressed message data and populate internal buffer.
- **Inputs:** Uninflated (compressed) message object.
- **Outputs/Return:** `true` on success, `false` on decompression failure.
- **Side effects:** Allocates temporary buffer, decompresses via `uncompress()`, calls `copyBufferFrom()`.
- **Calls:** `AIStreamBE()`, `uncompress()` (zlib), `copyBufferFrom()`.
- **Notes:** Reads uncompressed size (uint32 BE) from first 4 bytes of buffer; memory allocated via `auto_ptr`.

### BigChunkOfZippedDataMessage::deflate
- **Signature:** `UninflatedMessage* BigChunkOfZippedDataMessage::deflate() const`
- **Purpose:** Compress message data using zlib and return new `UninflatedMessage` with compressed payload.
- **Inputs:** None (uses internal buffer).
- **Outputs/Return:** Pointer to new `UninflatedMessage` (caller owns), or `NULL` on compression failure.
- **Side effects:** Allocates new message, computes compressed size as `length() * 105 / 100 + 12`, writes uncompressed length as uint32 BE prefix.
- **Calls:** `compress()` (zlib), `new UninflatedMessage()`, `AOStreamBE()`, `memcpy()`.
- **Notes:** If `length() == 0`, returns message with size 0; returns `NULL` if `compress()` fails; caller must `delete` result.

### AcceptJoinMessage::reallyDeflateTo / reallyInflateFrom
- **Signature:** `void reallyDeflateTo(AOStream& outputStream) const` / `bool reallyInflateFrom(AIStream& inputStream)`
- **Purpose:** Serialize/deserialize join acceptance (boolean + player info).
- **Inputs:** Stream, (inflate) or none (deflate).
- **Outputs/Return:** `true` on success (inflate).
- **Side effects:** Writes/reads boolean flag and deflates/inflates associated `NetPlayer`.
- **Calls:** Stream operators, `deflateNetPlayer()`/`inflateNetPlayer()`.

### CapabilitiesMessage / ChangeColorsMessage / ClientInfoMessage / HelloMessage / JoinerInfoMessage / NetworkChatMessage / ServerWarningMessage / TopologyMessage
- **Pattern:** All implement `reallyDeflateTo()` and `reallyInflateFrom()` pairs for respective message types.
- **CapabilitiesMessage:** Iterates over map of string keys and values; no length prefix (reads until stream end).
- **ChangeColorsMessage:** Trivial ΓÇö two int16 fields (color, team).
- **ClientInfoMessage:** Serializes stream ID, action, color, team, and name string.
- **HelloMessage:** Single version string.
- **JoinerInfoMessage:** Stream ID, Pascal name, version string, color, team.
- **NetworkChatMessage:** Sender ID, target, target ID, chat text (up to 1024 chars).
- **ServerWarningMessage:** Reason (uint16), warning string (up to 1024 chars).
- **TopologyMessage:** Complex ΓÇö serializes topology tag, player count, game settings, level name, and 16 `NetPlayer` structs via loop.

## Control Flow Notes
This file does not directly handle frame/render loops. It is invoked during **network initialization and game setup** when messages must be exchanged between server (gatherer) and clients (joiners):
1. **Hello** / **JoinerInfo** messages establish version and capabilities.
2. **Topology** messages distribute game state and player roster.
3. **Chat**, **ChangeColors**, **ServerWarning** messages maintain session state.
4. **BigChunkOfZippedDataMessage** subclasses (MapMessage, PhysicsMessage, LuaMessage) transfer large assets with compression.

Serialization (deflate) occurs before transmission; deserialization (inflate) occurs on receipt.

## External Dependencies
- **cseries.h** ΓÇö base platform/compiler compatibility layer
- **AStream.h** ΓÇö `AIStream`, `AOStream`, `AIStreamBE`, `AOStreamBE` classes for endian-aware serialization
- **network_messages.h** ΓÇö message class declarations (base classes like `SmallMessageHelper`, `BigChunkOfDataMessage`)
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `ClientChatInfo`, protocol constants
- **network_data_formats.h** ΓÇö network-safe struct packing/unpacking utilities
- **zlib.h** ΓÇö `compress()`, `uncompress()` for data compression
- **Message.h** (assumed) ΓÇö `UninflatedMessage` base class and message framework
