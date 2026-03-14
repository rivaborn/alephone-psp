# Source_Files/LibNAT/error.h
## File Purpose
Defines error and status codes for the LibNAT (Network Address Translation) library across socket, HTTP, SSDP, and UPnP subsystems. Provides a centralized error-code enumeration system where all LibNAT functions return error codes that can be printed to stderr via a utility function.

## Core Responsibilities
- Define success/failure status codes (OK, LNAT_ERROR)
- Define generic errors (memory allocation, parameter validation)
- Define socket-level errors (connection, send/recv timeouts, socket operations)
- Define HTTP protocol errors (header parsing, content-length, URL validation)
- Define SSDP discovery errors (header parsing, missing location tags)
- Define UPnP errors (description parsing, public IP discovery)
- Provide error-to-text printing via LNat_Print_Internal_Error()

## External Dependencies
- Implicitly uses standard C library (stderr output in LNat_Print_Internal_Error implementation)
- No explicit includes in this header

# Source_Files/LibNAT/http.h
## File Purpose
Header file declaring HTTP client functions for constructing and sending GET/POST requests to a network device (router). Provides APIs for message generation, header manipulation, and request execution. Part of the LibNAT UPnP/NAT library for router communication.

## Core Responsibilities
- Declare constructors/destructors for HTTP GET and POST message structures
- Provide header field manipulation (Request and Entity headers for POST messages)
- Declare HTTP GET and POST request execution functions
- Manage memory allocation/deallocation for HTTP messages and responses

## External Dependencies
- Standard C library (implied free() usage in comments)
- No includes visible; types are forward-declared to maintain encapsulation.

# Source_Files/LibNAT/libnat.h
## File Purpose
Public C API header for controlling UPnP-enabled Internet Gateway Devices. Provides functions for discovering IGDs on local networks, retrieving public IP addresses, and managing port mappings to enable NAT traversal for applications (e.g., file transfers, multiplayer connections).

## Core Responsibilities
- Declare the public API for UPnP IGD control operations
- Define the `UpnpController` structure to encapsulate discovered IGD connection details
- Provide device discovery, IP retrieval, and bidirectional port mapping operations
- Include error codes and diagnostic utilities via `error.h`
- Support compilation as both C and C++ (extern "C" guards)
- Define DLL export/import macros for Windows builds

## External Dependencies
- `error.h` ΓÇö defines error code constants (`OK`, `BAD_MALLOC`, `UPNP_*`, `SOCKET_*`, `HTTP_*`, `SSDP_*`, etc.) used as return values
- Platform headers (Windows): conditional `__declspec(dllexport/dllimport)` for DLL build support (currently disabled)
- Standard C library types: `char`, `int`, `short int`

# Source_Files/LibNAT/os.h
## File Purpose
OS abstraction header that provides a unified networking interface for both Unix and Windows systems. Implements compile-time platform detection and conditional macro aliasing to route socket operations to platform-specific implementations.

## Core Responsibilities
- Detect build platform (Unix vs Windows) via preprocessor conditionals
- Provide platform-agnostic socket operation macros (UDP/TCP setup, send, recv, close)
- Declare unified function prototypes for socket and utility operations
- Define opaque `OsSocket` type for cross-platform socket handles

## External Dependencies
- **Includes:** None (pure C declarations)
- **Preprocessor symbols used:** `_WIN32`, `WIN32`, `__CYGWIN__`, `__MINGW32__`, `__BORLANDC__` (platform detection)
- **Standard C types:** `int`, `char`, `short int` (built-in)

---

**Notes:**
- All socket operation functions are macros that conditionally alias to `LNat_Unix_*` or `LNat_Win_*` implementations.
- `OsSocket` is forward-declared as opaque; its layout is hidden from callers and defined in platform-specific source files.
- Function prototypes declare return codes as `int`; actual error semantics likely documented in companion implementation files.

# Source_Files/LibNAT/os_common.h
## File Purpose
Platform abstraction header for socket operations in the LibNAT library. Defines a unified interface for UDP and TCP communication across Unix and Windows, with platform-specific type definitions and common function declarations used by OS-specific implementations.

## Core Responsibilities
- Define platform-specific `OsSocket` opaque type (int socket for Unix, SOCKET for Windows)
- Include conditional OS-specific headers (Unix: fcntl, socket, netinet, arpa, netdb; Windows: windows.h, winsock.h)
- Declare UDP socket operations (send/receive to arbitrary hosts)
- Declare TCP socket operations (send/receive on connected sockets)
- Declare socket utility functions (address initialization, I/O readiness checks, local IP retrieval)
- Provide forward declarations for `sockaddr_in` and `hostent` structures

## External Dependencies
- **os.h**: platform detection (`OS_UNIX`, `OS_WIN`) and function interface macros
- **Unix headers** (conditional): fcntl.h, time.h, sys/types.h, sys/socket.h, sys/select.h, netinet/in.h, arpa/inet.h, netdb.h, stdlib.h, unistd.h, sys/time.h, string.h
- **Windows headers** (conditional): windows.h, winsock.h
- **Forward-declared structures**: `struct sockaddr_in` (netinet/in.h or winsock.h), `struct hostent` (netdb.h or winsock.h)

# Source_Files/LibNAT/ssdp.h
## File Purpose
Header file for SSDP (Simple Service Discovery Protocol) functionality in a NAT/UPnP library. Declares a single function to send SSDP discovery requests and receive router responses, typically used for UPnP device discovery.

## Core Responsibilities
- Declare SSDP service discovery interface
- Enable sending discovery requests to network devices
- Provide mechanism for receiving discovery responses from routers/UPnP devices

## External Dependencies
- Assumes standard C library `free()` for caller cleanup
- SSDP protocol implementation (defined elsewhere in LibNAT)

# Source_Files/LibNAT/utility.h
## File Purpose
Defines common utility constants and function declarations for the LibNAT library. Provides standardized size limits for HTTP/URL handling, port numbers, and C-string operations used throughout the project.

## Core Responsibilities
- Define standard C-string handling constants (null terminators)
- Specify constraints for port numbers (max value and string length)
- Define HTTP protocol constants (status codes, protocol prefix, default port)
- Define size limits for URL components (URLs, hostnames, resource paths)
- Declare string utility functions (case conversion)

## External Dependencies
- Standard C conventions only.
- No `#include` directives shown; expected to be included by other translation units that need these constants and the string utility function.
- Function implementation defined elsewhere (presumably `utility.c`).


