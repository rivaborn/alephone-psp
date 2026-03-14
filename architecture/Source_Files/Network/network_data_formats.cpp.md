# Source_Files/Network/network_data_formats.cpp

## File Purpose
Implements bidirectional serialization functions (netcpy overloads) that convert between platform-native and network-portable binary formats. Handles byte-order swapping and packing/unpacking of network packet structures for the Aleph One game engine's multiplayer layer, ensuring consistent data representation across different platforms and architectures.

## Core Responsibilities
- Convert NetPacketHeader between native and network byte order
- Convert NetPacket (ring buffer packets) between native and network formats, handling player action flags
- Swap byte order for uint32 arrays on little-endian platforms
- Convert NetDistributionPacket (broadcast packets) between formats
- Convert IPaddress (host/port pairs) to/from network byte order without swapping
- Convert network_audio_header structures between formats
- Ensure portable network serialization via stream-based packing/unpacking

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPacketHeader_NET | struct | Network-portable header (6 bytes: tag, sequence) |
| NetPacket_NET | struct | Network-portable ring packet (variable, ~40+ bytes with player action counts) |
| NetDistributionPacket_NET | struct | Network-portable distribution packet (6 bytes: type, player index, data size) |
| IPaddress_NET | struct | Network-portable address (6 bytes: 32-bit host, 16-bit port) |
| network_audio_header_NET | struct | Network-portable audio header (8 bytes: reserved, flags) |

## Global / File-Static State
None.

## Key Functions / Methods

### netcpy (NetPacketHeader_NET* ΓåÆ NetPacketHeader)
- Signature: `void netcpy(NetPacketHeader_NET* dest, const NetPacketHeader* src)`
- Purpose: Pack native header into network format
- Inputs: src (native NetPacketHeader pointer)
- Outputs/Return: dest (network format buffer)
- Side effects: Writes 6 bytes to dest->data
- Calls: `ValueToStream()` (from Packing.h)
- Notes: Uses stream-based packing; asserts exact 6-byte boundary

### netcpy (NetPacketHeader* ΓåÉ NetPacketHeader_NET)
- Signature: `void netcpy(NetPacketHeader* dest, const NetPacketHeader_NET* src)`
- Purpose: Unpack network format into native header
- Inputs: src (network format buffer)
- Outputs/Return: dest (native NetPacketHeader pointer)
- Side effects: Populates dest->tag and dest->sequence
- Calls: `StreamToValue()`
- Notes: Inverse of packing variant; asserts boundary

### netcpy (NetPacket_NET* ΓåÆ NetPacket)
- Signature: `void netcpy(NetPacket_NET* dest, const NetPacket* src)`
- Purpose: Pack native ring packet into network format
- Inputs: src (native packet with ring_packet_type, server_player_index, server_net_time, required_action_flags, action_flag_count array)
- Outputs/Return: dest (network format buffer)
- Side effects: Writes to dest->data; loops over MAXIMUM_NUMBER_OF_NETWORK_PLAYERS
- Calls: `ValueToStream()`, memcpy-like operations (loop)
- Notes: Packs scalar fields then array; first two fields packed as raw uint8; asserts boundary

### netcpy (NetPacket* ΓåÉ NetPacket_NET)
- Signature: `void netcpy(NetPacket* dest, const NetPacket_NET* src)`
- Purpose: Unpack network format into native ring packet
- Inputs: src (network format buffer)
- Outputs/Return: dest (native NetPacket pointer)
- Side effects: Populates all fields of dest
- Calls: `StreamToValue()`, loop iteration
- Notes: Inverse of packing; action_flag_count array unpacked per player

### netcpy (uint32* ΓåÉ uint32*, conditional endian swap)
- Signature: `void netcpy(uint32* dest, const uint32* src, size_t length)` (ALEPHONE_LITTLE_ENDIAN only)
- Purpose: Pack/unpack uint32 array with byte-order swap on little-endian platforms
- Inputs: src (uint32 array), length (byte count, must be multiple of 4)
- Outputs/Return: dest (byte-swapped or copied array)
- Side effects: Writes swapped data to dest; calls ListToStream() for byte conversion
- Calls: `ListToStream()`
- Notes: Only compiled on ALEPHONE_LITTLE_ENDIAN; on big-endian, header declares inline memcpy fallback

### netcpy (NetDistributionPacket_NET* ΓåÆ NetDistributionPacket) & vice versa
- Signature: Bidirectional (pack and unpack)
- Purpose: Convert distribution/broadcast packets between formats
- Inputs: src (6-byte payload: distribution_type, first_player_index, data_size)
- Outputs/Return: dest (6-byte network buffer or native struct)
- Side effects: Writes 6 bytes via ValueToStream or unpacks 6 bytes
- Calls: `ValueToStream()` / `StreamToValue()`
- Notes: Note in header states data segment not processed here; payload interpretation left to caller

### netcpy (IPaddress_NET* Γåö IPaddress)
- Signature: Bidirectional (pack and unpack)
- Purpose: Convert IP address (host + port) to/from network format
- Inputs: src (host=uint32, port=uint16)
- Outputs/Return: dest (6-byte network buffer or native IPaddress struct)
- Side effects: Raw memcpy of host (4 bytes) and port (2 bytes); no endian swap
- Calls: `memcpy()` (direct, not via ValueToStream)
- Notes: **Important invariant**: host and port are already in network byte order; no byte-swapping performed

### netcpy (network_audio_header_NET* Γåö network_audio_header)
- Signature: Bidirectional (pack and unpack)
- Purpose: Convert audio header (reserved + flags) between formats
- Inputs: src (mReserved, mFlags)
- Outputs/Return: dest (8-byte network buffer or native struct)
- Side effects: Writes/reads 8 bytes via ValueToStream/StreamToValue
- Calls: `ValueToStream()` / `StreamToValue()`
- Notes: Asserts 8-byte boundary

## Control Flow Notes
This file is invoked during network packet serialization (outbound) and deserialization (inbound). It sits between the game engine's native data structures (which may have padding and endianness mismatches) and the on-wire format. Typical call path: game code ΓåÆ pack/unpack (this file) ΓåÆ socket I/O.

The file is conditionally compiled only when DISABLE_NETWORKING is not defined. Endian handling is compile-time determined via ALEPHONE_LITTLE_ENDIAN macro.

## External Dependencies
- **Packing.h**: Provides `ValueToStream()`, `StreamToValue()`, `ListToStream()` for byte-order-aware serialization
- **network_data_formats.h**: Declarations of all netcpy overloads and _NET struct definitions
- **network.h, network_private.h**: Definitions of native network structures (NetPacketHeader, NetPacket, etc.)
- **network_audio_shared.h**: Definition of network_audio_header
- **cseries.h**: Provides ALEPHONE_LITTLE_ENDIAN macro and stdint types (uint8, uint16, uint32, int32, int16)
