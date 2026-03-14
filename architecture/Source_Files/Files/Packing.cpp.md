# Source_Files/Files/Packing.cpp

## File Purpose
Implements byte-order conversion functions for serializing/deserializing game data. Converts between native in-memory values (int16, uint16, int32, uint32) and byte streams in either big-endian or little-endian format. Core infrastructure for Marathon's packed file format handling.

## Core Responsibilities
- Read values from byte streams in big-endian format (StreamToValueBE)
- Read values from byte streams in little-endian format (StreamToValueLE)
- Write values to byte streams in big-endian format (ValueToStreamBE)
- Write values to byte streams in little-endian format (ValueToStreamLE)
- Support signed and unsigned 16-bit and 32-bit integers
- Automatically advance stream pointers during read/write operations
- Handle sign-extension when converting from unsigned to signed representations

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamToValueBE (uint16 variant)
- Signature: `void StreamToValueBE(uint8* &Stream, uint16 &Value)`
- Purpose: Extract a 16-bit big-endian value from a byte stream
- Inputs: Stream pointer (by reference); output value reference
- Outputs: Value parameter populated; Stream advanced by 2 bytes
- Side effects: Modifies Stream pointer
- Calls: None (direct byte extraction)
- Notes: Reads byte0 (MSB), byte1 (LSB); combines as `(Byte0 << 8) | Byte1`

### StreamToValueBE (int16 variant)
- Signature: `void StreamToValueBE(uint8* &Stream, int16 &Value)`
- Purpose: Extract a signed 16-bit big-endian value
- Inputs: Stream pointer, output reference
- Outputs: Value; Stream advanced
- Side effects: Advances Stream pointer
- Calls: `StreamToValueBE(uint16)` for unsigned extraction, then casts result

### StreamToValueBE (uint32 & int32 variants)
- Signature: `void StreamToValueBE(uint8* &Stream, uint32 &Value)` and signed variant
- Purpose: Extract 32-bit values in big-endian order
- Inputs: Stream pointer, output reference
- Outputs: Value; Stream advanced by 4 bytes
- Side effects: Advances Stream pointer
- Calls: Self (int32 delegates to uint32)
- Notes: Combines four bytes as `(Byte0 << 24) | (Byte1 << 16) | (Byte2 << 8) | Byte3`

### ValueToStreamBE / StreamToValueLE / ValueToStreamLE
- Symmetric counterparts: write instead of read, or little-endian instead of big-endian
- All follow the same overload structure (uint16, int16, uint32, int32)
- Little-endian variants reverse byte order: LSB written/read first

## Control Flow Notes
These are utility functions called during file I/O operations (serialization/deserialization of game assets). The header file (`Packing.h`) defines macro aliases (`StreamToValue`, `ValueToStream`) that select the big-endian or little-endian variant at compile time based on `PACKED_DATA_IS_BIG_ENDIAN` or `PACKED_DATA_IS_LITTLE_ENDIAN`. Functions are non-inlined (moved to .cpp per header comment dated 2002) to avoid compiler inlining issues.

## External Dependencies
- `cseries.h`: Provides base type definitions (uint8, int16, uint16, int32, uint32)
- `Packing.h`: Function declarations and macro configuration
- Implicit: SDL.h, SDL_byteorder.h (included transitively via cseries.h)
