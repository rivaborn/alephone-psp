# Source_Files/Files/AStream.cpp

## File Purpose
Implements type-safe binary serialization/deserialization with explicit Big Endian and Little Endian byte ordering. Provides input/output stream classes for network protocols and persistent storage, replacing less clear endianness handling from AlephOne's Packing system.

## Core Responsibilities
- Deserialization of 8/16/32-bit signed and unsigned integers with endianness control
- Serialization of integers to byte streams with specified byte order
- Raw byte reading/writing operations with bounds checking
- Stream state management (fail/bad bits) and exception-based error signaling
- Support for chaining operations via operator>> and operator<< return values

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AStream::basic_astream<T> | template class | Base stream with state, bounds, and position tracking |
| AStream::failure | exception class | Custom exception with dynamic message storage for serialization errors |
| AIStream | class | Abstract input stream base for deserialization |
| AIStreamBE | class | Big Endian input stream (16/32-bit multi-byte reading) |
| AIStreamLE | class | Little Endian input stream (16/32-bit multi-byte reading) |
| AOStream | class | Abstract output stream base for serialization |
| AOStreamBE | class | Big Endian output stream (16/32-bit multi-byte writing) |
| AOStreamLE | class | Little Endian output stream (16/32-bit multi-byte writing) |

## Global / File-Static State
None.

## Key Functions / Methods

### AIStream::operator>>(uint8 &value)
- Signature: `AIStream& operator>>(uint8 &value) throw(AStream::failure)`
- Purpose: Extract unsigned 8-bit integer from stream
- Inputs: Reference to uint8 receiving the value
- Outputs/Return: Reference to this stream (for chaining)
- Side effects: Advances stream position by 1 if bounds check passes; sets failbit if overflow
- Calls: bound_check(1)
- Notes: No endianness handling needed for single bytes

### AIStreamBE::operator>>(uint16 &value)
- Signature: `AIStream& operator>>(uint16 &value) throw(AStream::failure)`
- Purpose: Extract Big Endian 16-bit unsigned integer
- Inputs: Reference to uint16 receiving the value
- Outputs/Return: Reference to this stream
- Side effects: Reads 2 bytes, advances position
- Calls: bound_check(2)
- Notes: Byte order: (Byte0 << 8) | Byte1 (MSB first)

### AIStreamBE::operator>>(uint32 &value)
- Signature: `AIStream& operator>>(uint32 &value) throw(AStream::failure)`
- Purpose: Extract Big Endian 32-bit unsigned integer
- Inputs: Reference to uint32 receiving the value
- Outputs/Return: Reference to this stream
- Side effects: Reads 4 bytes, advances position
- Calls: bound_check(4)
- Notes: Byte order: (Byte0 << 24) | (Byte1 << 16) | (Byte2 << 8) | Byte3

### AIStreamLE::operator>>(uint16 &value)
- Signature: `AIStream& operator>>(uint16 &value) throw(AStream::failure)`
- Purpose: Extract Little Endian 16-bit unsigned integer
- Inputs: Reference to uint16 receiving the value
- Outputs/Return: Reference to this stream
- Side effects: Reads 2 bytes, advances position
- Calls: bound_check(2)
- Notes: Byte order: (Byte1 << 8) | Byte0 (LSB first)

### AIStreamLE::operator>>(uint32 &value)
- Signature: `AIStream& operator>>(uint32 &value) throw(AStream::failure)`
- Purpose: Extract Little Endian 32-bit unsigned integer
- Inputs: Reference to uint32 receiving the value
- Outputs/Return: Reference to this stream
- Side effects: Reads 4 bytes, advances position
- Calls: bound_check(4)
- Notes: Byte order: (Byte3 << 24) | (Byte2 << 16) | (Byte1 << 8) | Byte0

### AOStream::operator<<(uint8 value)
- Signature: `AOStream& operator<<(uint8 value) throw(AStream::failure)`
- Purpose: Write unsigned 8-bit integer to stream
- Inputs: uint8 value to write
- Outputs/Return: Reference to this stream
- Side effects: Writes 1 byte, advances position if bounds check passes
- Calls: bound_check(1)

### AOStreamBE::operator<<(uint16 value)
- Signature: `AOStream& operator<<(uint16 value) throw(AStream::failure)`
- Purpose: Write Big Endian 16-bit unsigned integer
- Inputs: uint16 value to write
- Outputs/Return: Reference to this stream
- Side effects: Writes 2 bytes in Big Endian order
- Calls: bound_check(2)
- Notes: Writes MSB first: (value >> 8), then value

### AOStreamBE::operator<<(uint32 value)
- Signature: `AOStream& operator<<(uint32 value) throw(AStream::failure)`
- Purpose: Write Big Endian 32-bit unsigned integer
- Inputs: uint32 value to write
- Outputs/Return: Reference to this stream
- Side effects: Writes 4 bytes in Big Endian order
- Calls: bound_check(4)
- Notes: Writes in order: (value >> 24), (value >> 16), (value >> 8), value

### AOStreamLE::operator<<(uint16 value)
- Signature: `AOStream& operator<<(uint16 value) throw(AStream::failure)`
- Purpose: Write Little Endian 16-bit unsigned integer
- Inputs: uint16 value to write
- Outputs/Return: Reference to this stream
- Side effects: Writes 2 bytes in Little Endian order
- Calls: bound_check(2)
- Notes: Writes LSB first: value, then (value >> 8)

### AOStreamLE::operator<<(uint32 value)
- Signature: `AOStream& operator<<(uint32 value) throw(AStream::failure)`
- Purpose: Write Little Endian 32-bit unsigned integer
- Inputs: uint32 value to write
- Outputs/Return: Reference to this stream
- Side effects: Writes 4 bytes in Little Endian order
- Calls: bound_check(4)
- Notes: Writes in order: value, (value >> 8), (value >> 16), (value >> 24)

### AStream::basic_astream<T>::bound_check(uint32 delta)
- Signature: `bool bound_check(uint32 delta) throw(AStream::failure)`
- Purpose: Verify that advancing by delta bytes will not exceed stream bounds
- Inputs: Number of bytes to validate for
- Outputs/Return: true if bounds valid and not in fail state; false otherwise
- Side effects: Sets failbit if overflow detected; throws failure exception if exceptions enabled
- Calls: setstate(), exceptions()
- Notes: Prevents buffer overruns; critical safety gate for all I/O operations

### AStream::failure (exception class)
- **Constructor** `failure(const std::string& str)`: Allocates and copies error message via strdup()
- **Copy constructor** `failure(const failure &f)`: Deep copies message if present
- **Destructor** `~failure()`: Frees message memory and nullifies pointer
- **what()**: Returns message C-string; overrides std::exception interface

## Control Flow Notes
This is a utility library, not part of engine init/frame/update/render cycles. Typical usage: (1) construct appropriate stream type (AIStreamBE/LE for input, AOStreamBE/LE for output) with a buffer; (2) chain operator>> or operator<< calls to read/write typed values; (3) bounds checking automatically validates each operation; (4) exception or failbit state indicates serialization failures.

The class hierarchy uses inheritance to separate concerns: base classes (AIStream/AOStream) handle 8-bit I/O and interface definition, derived classes (BE/LE variants) specialize endianness for 16/32-bit operations.

## External Dependencies
- `<string>` ΓÇö std::string for exception messages
- `<exception>` ΓÇö std::exception base class
- `<string.h>` ΓÇö memcpy() for bulk byte operations, strdup()/free() for message storage
- `"cstypes.h"` ΓÇö Type definitions (uint8, int8, uint16, int16, uint32, int32)
- Uses `std::` namespace
