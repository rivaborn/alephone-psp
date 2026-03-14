# Source_Files/Network/network_data_formats.h

## File Purpose
Defines platform-independent network packet structures and serialization/deserialization functions. Provides a conversion layer between native C structs (used internally) and fixed-format "_NET" wire structures (transmitted over network), ensuring consistent byte ordering and padding across all platforms.

## Core Responsibilities
- Define fixed-size wire-format packet structures (`*_NET` variants) with byte-array representation
- Declare `netcpy()` conversion functions for bidirectional marshaling between native and wire formats
- Handle endianness conversions (byte-swapping on little-endian architectures)
- Ensure consistent network protocol serialization across platforms (Windows, Mac, Linux)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NetPacketHeader_NET` | struct | Wire format (6 bytes) for ring packet headers (tag + sequence number) |
| `NetPacket_NET` | struct | Wire format for game action flag ring packets |
| `NetDistributionPacket_NET` | struct | Wire format (6 bytes) for custom distribution packets |
| `IPaddress_NET` | struct | Wire format (6 bytes) for IP address + port pairs |
| `network_audio_header_NET` | struct | Wire format (8 bytes) for network audio frame headers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SIZEOF_NetPacketHeader` | `const int` | file-static | Size of wire packet header: 6 bytes |
| `SIZEOF_NetPacket` | `const int` | file-static | Size of wire ring packet |
| `SIZEOF_NetDistributionPacket` | `const int` | file-static | Size of wire distribution packet: 6 bytes |
| `SIZEOF_IPaddress` | `const int` | file-static | Size of wire IP address: 6 bytes |
| `SIZEOF_network_audio_header` | `const int` | file-static | Size of wire audio header: 8 bytes |

## Key Functions / Methods

### netcpy (NetPacketHeader)
- Signature: `void netcpy(NetPacketHeader_NET* dest, const NetPacketHeader* src)` and reverse
- Purpose: Convert packet header between native and wire format
- Inputs: Source struct (native or wire)
- Outputs/Return: Destination struct written in place
- Side effects: Memory write to destination pointer
- Calls: Defined elsewhere in network_data_formats.cpp

### netcpy (NetPacket)
- Signature: `void netcpy(NetPacket_NET* dest, const NetPacket* src)` and reverse
- Purpose: Convert ring packet data (tag, timestamps, player indices, action flags) between formats
- Inputs: Source struct
- Outputs/Return: Destination struct written in place
- Side effects: Memory write
- Notes: Action flag array is left unmarshed (handled separately via uint32 overload)

### netcpy (uint32 action flags)
- Signature: `void netcpy(uint32* dest, const uint32* src, size_t length)`
- Purpose: Endian-convert action flag arrays (length in bytes)
- Inputs: Source pointer, destination pointer, byte count
- Outputs/Return: Byte-swapped copy written to destination
- Side effects: On big-endian, delegates to `memcpy`; on little-endian, performs byte-swap
- Notes: Bidirectional; same function used for wireΓåÆnative and nativeΓåÆwire conversion

### netcpy (NetDistributionPacket)
- Signature: `void netcpy(NetDistributionPacket_NET* dest, const NetDistributionPacket* src)` and reverse
- Purpose: Marshal distribution packet headers (type, player index, data size)
- Notes: Payload data field is opaque; caller responsible for its marshaling

### netcpy (IPaddress)
- Signature: `void netcpy(IPaddress_NET* dest, const IPaddress* src)` and reverse
- Purpose: Convert IP address structures
- Notes: Host and port fields are **not** byte-swapped; always use network byte order

### netcpy (network_audio_header)
- Signature: `void netcpy(network_audio_header_NET* dest, const network_audio_header* src)` and reverse
- Purpose: Convert audio frame headers (reserved field + flags) between formats
- Inputs: Source struct
- Outputs/Return: Destination struct

## Control Flow Notes
This file operates at the **serialization boundary** of the network subsystem. When ring packets or distribution packets are sent, native structures are converted to `_NET` format via `netcpy()` before transmission. On receipt, `_NET` structures are converted back to native format. The header is included by network implementation files (e.g., network_data_formats.cpp, ring protocol handler) that perform the actual marshal/unmarshal operations during frame updates.

## External Dependencies
- **cseries.h** ΓÇö provides `ALEPHONE_LITTLE_ENDIAN` conditional definition; includes SDL byte-order macros
- **network.h** ΓÇö public API; defines `MAXIMUM_NUMBER_OF_NETWORK_PLAYERS` constant
- **network_private.h** ΓÇö defines native struct types (`NetPacketHeader`, `NetPacket`, `NetDistributionPacket`, `IPaddress`)
- **network_audio_shared.h** ΓÇö defines `network_audio_header` native struct
- **cstring.h** (implicit) ΓÇö `memcpy()` for big-endian fast path in uint32 overload
