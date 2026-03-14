# Source_Files/Files/AStream.h

## File Purpose
Serialization/deserialization abstraction providing template-based binary stream I/O with explicit endianness handling. Replaces the less clear `Packing` classes from AlephOne with better type safety and clearer endian semantics. Supports bounds checking, exception handling, and state management similar to `std::iostream`.

## Core Responsibilities
- Define template-based stream base class (`basic_astream<T>`) for buffer-based I/O
- Provide abstract input stream (`AIStream`) with extraction operators (`operator>>`)
- Provide abstract output stream (`AOStream`) with insertion operators (`operator<<`)
- Implement big-endian and little-endian variants (`AIStreamBE`, `AIStreamLE`, `AOStreamBE`, `AOStreamLE`)
- Manage I/O state (good/bad/fail bits) and exception masking
- Enforce bounds checking and raise exceptions on buffer overflow
- Support reading/writing primitive types (int8, uint8, int16, uint16, int32, uint32) and raw byte arrays

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `_Aiostate` | enum | I/O state flag enumeration |
| `iostate` | typedef | Type alias for state management |
| `failure` | class | Exception thrown on stream errors |
| `basic_astream<T>` | class template | Generic stream base with state and bounds tracking |
| `AIStream` | abstract class | Input stream interface; deserialization |
| `AIStreamBE` | class | Big-endian input stream (concrete) |
| `AIStreamLE` | class | Little-endian input stream (concrete) |
| `AOStream` | abstract class | Output stream interface; serialization |
| `AOStreamBE` | class | Big-endian output stream (concrete) |
| `AOStreamLE` | class | Little-endian output stream (concrete) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_S_badbit` | static const short | namespace AStream | Bit flag for bad state |
| `_S_failbit` | static const short | namespace AStream | Bit flag for fail state |
| `badbit` | static const iostate | namespace AStream | I/O error flag constant |
| `failbit` | static const iostate | namespace AStream | Logical failure flag constant |
| `goodbit` | static const iostate | namespace AStream | No errors flag constant |

## Key Functions / Methods

### basic_astream<T>::basic_astream (constructor)
- **Signature:** `basic_astream(T* __stream, uint32 __length, uint32 __offset)`
- **Purpose:** Initialize stream with buffer, length, and starting offset.
- **Inputs:** Buffer pointer, buffer length in elements, read/write offset in elements.
- **Outputs/Return:** None (constructor).
- **Side effects:** Sets `_M_stream_pos`, `_M_state`, `_M_exception`; sets badbit if offset exceeds buffer.
- **Calls:** None directly.
- **Notes:** Assumes `__offset <= __length`; default exception mask is `failbit`.

### basic_astream<T>::bound_check
- **Signature:** `bool bound_check(uint32 __delta) throw(AStream::failure)`
- **Purpose:** Verify that reading/writing `__delta` bytes from current position won't overflow.
- **Inputs:** Number of bytes to check.
- **Outputs/Return:** Boolean (true if safe); throws `failure` if exceptions enabled and bounds violated.
- **Side effects:** May set failbit or badbit on stream state.
- **Calls:** (implementation not visible; likely in .cpp).
- **Notes:** Throws only if `_M_exception` is set to catch the violation.

### AIStream::operator>>, AOStream::operator<<
- **Signatures:** `AIStream& operator>>(T &__value) throw(AStream::failure)` (8 variants for int8, uint8, int16, uint16, int32, uint32, bool, and arrays).
- **Purpose:** Extract/insert typed values with endian conversion; non-virtual for 8-bit types, virtual for 16/32-bit.
- **Inputs:** Reference to variable to fill (input) or value to write (output).
- **Outputs/Return:** Reference to stream (for chaining).
- **Side effects:** Advances `_M_stream_pos`; may set state bits; throws if bounds exceeded or exception mask set.
- **Calls:** `bound_check()` (inferred); endian-specific subclasses call parent for 8-bit types.
- **Notes:** Endian variants (BE/LE) override 16/32-bit operators to handle byte swapping.

### AIStream::read, AOStream::write (template)
- **Signature:** `T& read(T* __list, uint32 __count) throw(AStream::failure)` (similar for write).
- **Purpose:** Read or write an array of typed objects using overloaded `operator>>` or `operator<<`.
- **Inputs:** Pointer to array, count of elements.
- **Outputs/Return:** Reference to stream.
- **Side effects:** Calls `operator>>` or `operator<<` `__count` times; advances position by `count * sizeof(T)`.
- **Calls:** `operator>>` / `operator<<` in loop.
- **Notes:** Allows friendly overload resolution for custom types; non-template overloads exist for raw char/unsigned char/signed char.

### AIStream::tellg, AIStream::maxg, AOStream::tellp, AOStream::maxp
- **Signature:** `uint32 tellg() const`, `uint32 maxg() const`, etc.
- **Purpose:** Query current read/write position and buffer bounds.
- **Inputs:** None.
- **Outputs/Return:** Position or max position as uint32 offset from buffer start.
- **Side effects:** None.
- **Calls:** `tell_pos()`, `max_pos()` (inline helpers).
- **Notes:** Mirrors `std::iostream` conventions (g = get/input, p = put/output).

### AIStream::ignore, AOStream::ignore
- **Signature:** `AIStream& ignore(uint32 __count) throw(AStream::failure)`, similar for AOStream.
- **Purpose:** Skip `__count` bytes forward without reading/writing data.
- **Inputs:** Number of bytes to skip.
- **Outputs/Return:** Reference to stream.
- **Side effects:** Advances position; may fail on bounds overflow.
- **Calls:** (implementation not visible).
- **Notes:** Useful for aligning to boundaries or skipping padding.

## Control Flow Notes
This is a header file defining an I/O abstraction layer. Control flow depends on user code instantiating `AIStreamBE`/`AIStreamLE`/`AOStreamBE`/`AOStreamLE` with a buffer and calling `operator>>` / `operator<<` in sequence. State is checked post-operation via `good()`, `fail()`, `bad()`, or via exceptions if the exception mask is set. Endianness is fixed at instantiation; runtime endian switching is not supported.

## External Dependencies
- `#include <string>` ΓÇö for `std::string` in `failure` exception class.
- `#include <exception>` ΓÇö for `std::exception` base class.
- `#include "cstypes.h"` ΓÇö defines `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`.
