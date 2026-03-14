# Source_Files/Network/SSLP_Protocol.h

## File Purpose
Protocol definition header for SSLP (Simple Service Location Protocol), a UDP-based service discovery mechanism for locating network game servers. Defines packet structure, constants, and operation semantics for the Aleph One game engine's network player discovery system.

## Core Responsibilities
- Define SSLP protocol constants (magic, version, message types, port)
- Specify the wire format of SSLP_Packet for cross-platform compatibility
- Document protocol operation (FIND/HAVE/LOST message flow)
- Define service type/name string length constraints
- Ensure compiler-independent struct packing for network serialization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SSLP_Packet | struct | Network packet containing service discovery message (FIND/HAVE/LOST) with magic, version, message type, service port, and variable service type/name strings |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| SSLP_PORT | #define | global | UDP port for SSLP protocol (default 15367) |
| SSLPP_MAGIC | #define | global | Protocol magic number ("sslp" in network byte order: 0x73736c70) |
| SSLPP_VERSION | #define | global | Protocol version (currently 1) |
| SSLPP_MESSAGE_FIND | #define | global | Message type constant for service discovery request |
| SSLPP_MESSAGE_HAVE | #define | global | Message type constant for service advertisement |
| SSLPP_MESSAGE_LOST | #define | global | Message type constant for service unavailability notice |
| SSLP_MAX_TYPE_LENGTH | #define | global | Max length of service_type field (32 bytes) |
| SSLP_MAX_NAME_LENGTH | #define | global | Max length of service_name field (32 bytes) |
| SIZEOF_SSLP_Packet | const int | global | Calculated packed size (80 bytes) to avoid compiler-dependent alignment |

## Key Functions / Methods
NoneΓÇöthis is a protocol header containing only data structure and constant definitions.

## Control Flow Notes
Not applicable; this header defines static protocol format, not dynamic flow. Usage occurs in SSLP service implementation files that send/receive SSLP_Packet datagrams on SSLP_PORT via UDP broadcast or unicast.

## External Dependencies
- `<SDL_net.h>` ΓÇô SDL cross-platform IP networking library (Uint32, Uint16 types)

---

**Notes:**
- Packet uses C-string semantics for service_type/name fields; comparison/copying stops at first `\0`.
- NULL termination is *not* required if all SSLP_MAX_*_LENGTH bytes are used.
- All numeric fields in wire format are network (big-endian) byte order.
- Magic and version fields catch protocol mismatches; message type field determines packet role (FIND/HAVE/LOST).
