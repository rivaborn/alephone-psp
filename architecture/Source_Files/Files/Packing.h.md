# Source_Files/Files/Packing.h

## File Purpose
Provides utility functions for serializing and deserializing numerical values and raw byte data between native memory layouts and packed big-endian or little-endian byte streams. Used throughout the Marathon series game engine to handle cross-platform data format conversion without relying on compiler-generated padding.

## Core Responsibilities
- Serialize/deserialize single numerical values (16-bit and 32-bit integers, signed and unsigned) to/from byte streams
- Serialize/deserialize arrays of numerical values maintaining stream pointer advancement
- Copy arbitrary byte blocks to/from streams with stream pointer management
- Abstract endianness conversion (big-endian vs. little-endian) via preprocessor macros
- Provide a consistent API where packing and unpacking routines are syntactically similar

## Key Types / Data Structures
None.

## Global / File-Static State
None. Endianness behavior is controlled via preprocessor defines:
| Define | Type | Scope | Purpose |
|--------|------|-------|---------|
| PACKED_DATA_IS_BIG_ENDIAN | Preprocessor flag | Global | Selects big-endian unpacking/packing function variants (default) |
| PACKED_DATA_IS_LITTLE_ENDIAN | Preprocessor flag | Global | Selects little-endian unpacking/packing function variants (mutually exclusive) |

## Key Functions / Methods

### StreamToValue (extern overloads for uint16, int16, uint32, int32)
- **Signature:** `extern void StreamToValue(uint8* &Stream, T &Value);`
- **Purpose:** Unpack a single numerical value from a byte stream, converting from packed endianness to native.
- **Inputs:** Stream pointer (by reference), numerical value type template parameter.
- **Outputs/Return:** Updates `Value` with unpacked data; advances `Stream` pointer by type size (2 or 4 bytes).
- **Side effects:** Modifies stream pointer; reads from stream.
- **Calls:** Resolves to `StreamToValueBE` or `StreamToValueLE` at preprocessing time (implementation in Packing.cpp).
- **Notes:** Type-overloaded for each supported integer type. Actual implementation moved to .cpp to avoid inlining issues.

### ValueToStream (extern overloads for uint16, int16, uint32, int32)
- **Signature:** `extern void ValueToStream(uint8* &Stream, T Value);`
- **Purpose:** Pack a single numerical value into a byte stream, converting from native endianness to packed format.
- **Inputs:** Stream pointer (by reference), numerical value.
- **Outputs/Return:** Advances `Stream` pointer by type size (2 or 4 bytes).
- **Side effects:** Modifies stream pointer; writes to stream.
- **Calls:** Resolves to `ValueToStreamBE` or `ValueToStreamLE` at preprocessing time (implementation in Packing.cpp).
- **Notes:** Type-overloaded for each supported integer type.

### StreamToList (template)
- **Signature:** `template<class T> inline static void StreamToList(uint8* &Stream, T* List, size_t Count);`
- **Purpose:** Unpack an array of numerical values from a stream by repeatedly calling `StreamToValue`.
- **Inputs:** Stream pointer, destination array pointer, element count.
- **Outputs/Return:** Populates `List` array; advances stream pointer by `Count * sizeof(T)`.
- **Side effects:** Modifies stream pointer and array memory.
- **Calls:** `StreamToValue` for each element.
- **Notes:** Inlined; template instantiation for each type used.

### ListToStream (template)
- **Signature:** `template<class T> inline static void ListToStream(uint8* &Stream, T* List, size_t Count);`
- **Purpose:** Pack an array of numerical values into a stream by repeatedly calling `ValueToStream`.
- **Inputs:** Stream pointer, source array pointer, element count.
- **Outputs/Return:** Advances stream pointer by `Count * sizeof(T)`.
- **Side effects:** Modifies stream pointer; writes to stream.
- **Calls:** `ValueToStream` for each element.
- **Notes:** Inlined; template instantiation for each type used.

### StreamToBytes (inline)
- **Signature:** `inline static void StreamToBytes(uint8* &Stream, void* Bytes, size_t Count);`
- **Purpose:** Copy `Count` raw bytes from stream into memory without endianness conversion.
- **Inputs:** Stream pointer, destination byte buffer, byte count.
- **Outputs/Return:** Advances stream pointer by `Count`.
- **Side effects:** Modifies stream pointer and destination memory.
- **Calls:** `memcpy`.
- **Notes:** No type information; blind byte copy.

### BytesToStream (inline)
- **Signature:** `inline static void BytesToStream(uint8* &Stream, const void* Bytes, size_t Count);`
- **Purpose:** Copy `Count` raw bytes into stream without endianness conversion.
- **Inputs:** Stream pointer, source byte buffer, byte count.
- **Outputs/Return:** Advances stream pointer by `Count`.
- **Side effects:** Modifies stream pointer; writes to stream.
- **Calls:** `memcpy`.
- **Notes:** No type information; blind byte copy.

## Control Flow Notes
This is a utility header invoked on-demand during data I/O operations (file loading, network serialization, etc.), not part of the main frame loop. The stream pointer pattern allows sequential reading/writing of heterogeneous data layouts by maintaining a stateful position through the byte buffer.

## External Dependencies
- `memcpy` (from `<string.h>`, included but commented out; likely included elsewhere)
- Built-in integer types: `uint8`, `int16`, `uint16`, `int32`, `uint32`
- Actual implementations in `Packing.cpp` (moved there to avoid compiler inlining issues)
