# Subsystem Overview

## Purpose

LibNAT is a UPnP/NAT traversal library that enables applications to discover Internet Gateway Devices (routers) on local networks, retrieve public IP addresses, and manage port mappings for NAT traversal. It abstracts socket operations across Unix and Windows platforms via HTTP and SSDP protocols to facilitate network connectivity in environments where the application cannot directly receive inbound connections.

## Key Files

| File | Role |
|------|------|
| libnat.h | Public C API for UPnP IGD control, device discovery, IP retrieval, and port mapping operations |
| error.h | Centralized error and status codes (socket, HTTP, SSDP, UPnP); error-to-text printing utility |
| http.h | HTTP client API for constructing and sending GET/POST requests to routers |
| ssdp.h | SSDP discovery protocol interface for locating UPnP devices on local networks |
| os.h | Platform detection and socket operation macro abstraction (Unix vs. Windows) |
| os_common.h | Platform-specific socket type definitions and unified socket function declarations |
| utility.h | Standard constants and utility functions (C-strings, port numbers, HTTP, URL size limits) |

## Core Responsibilities

- Discover Internet Gateway Devices via SSDP multicast broadcasts on local networks
- Retrieve the public IP address from discovered IGDs
- Create and delete bidirectional port mappings via UPnP protocol to enable inbound application connectivity
- Abstract socket operations (UDP/TCP send/recv, connection setup, I/O readiness) across Unix and Windows at compile time
- Parse HTTP responses and construct HTTP request messages for router UPnP service communication
- Provide centralized error code enumeration and error-to-text diagnostic utilities
- Maintain platform-specific socket encapsulation via opaque `OsSocket` type

## Key Interfaces & Data Flow

**Exposes to callers:**
- `UpnpController` structure encapsulating discovered IGD connection details
- Public functions for device discovery, public IP retrieval, and port mapping operations
- Error code constants and `LNat_Print_Internal_Error()` diagnostic function
- Extern "C" guards for C++ compatibility; Windows DLL export/import macros

**Consumes from subsystems:**
- Standard C library (malloc/free, stderr I/O)
- Platform-specific headers: Unix (fcntl.h, sys/socket.h, netinet/in.h, arpa/inet.h, netdb.h); Windows (windows.h, winsock.h)
- Preprocessor platform detection macros (`_WIN32`, `__CYGWIN__`, `__MINGW32__`)

## Runtime Role

LibNAT participates during application initialization to discover local routers and establish port mappings before network services activate. Public IP addresses may be retrieved for connection string generation or diagnostics. No frame-level or explicit shutdown hooks are evident in the provided headers.

## Notable Implementation Details

- **Platform abstraction**: `os.h` uses preprocessor conditionals to alias socket operation macros to `LNat_Unix_*` or `LNat_Win_*` implementations; `OsSocket` is opaque and defined per-platform in source files.
- **Error semantics**: All LibNAT functions return `int` error codes; callers must check return values and may call `LNat_Print_Internal_Error()` to print diagnostics to stderr.
- **Protocol layering**: HTTP client sits atop the OS socket abstraction layer; SSDP discovery bridges local network discovery to HTTP communication with UPnP services.
- **Memory management**: Callers are responsible for freeing HTTP message and response structures via standard C `free()`.
- **Windows build support**: DLL export/import macros (`__declspec`) are defined but currently disabled.
