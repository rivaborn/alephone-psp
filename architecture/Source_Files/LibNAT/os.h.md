# Source_Files/LibNAT/os.h

## File Purpose
OS abstraction header that provides a unified networking interface for both Unix and Windows systems. Implements compile-time platform detection and conditional macro aliasing to route socket operations to platform-specific implementations.

## Core Responsibilities
- Detect build platform (Unix vs Windows) via preprocessor conditionals
- Provide platform-agnostic socket operation macros (UDP/TCP setup, send, recv, close)
- Declare unified function prototypes for socket and utility operations
- Define opaque `OsSocket` type for cross-platform socket handles

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OsSocket` | struct (opaque) | Opaque handle for platform-specific socket implementation |

## Global / File-Static State
None.

## Key Functions / Methods
These are function declarations only; implementations reside in platform-specific source files (`os.c` variants).

### LNat_Os_Socket_Udp_Setup
- **Signature:** `int LNat_Os_Socket_Udp_Setup(OsSocket ** s)`
- **Purpose:** Allocate and initialize a UDP socket
- **Inputs:** Pointer to socket handle pointer
- **Outputs/Return:** Integer status code; socket initialized via `*s`
- **Calls:** Routed to `LNat_Unix_Socket_Udp_Setup` (Unix) or `LNat_Win_Socket_Udp_Setup` (Windows) via macro

### LNat_Os_Socket_Udp_Send / Recv
- **Signature:** `int LNat_Os_Socket_Udp_Send/Recv(OsSocket *, host_addr, port, buf, amt, [amt_sent/recv], [timeout])`
- **Purpose:** Send/receive UDP datagrams with optional timeout
- **Inputs:** Socket, destination host, port, buffer, byte count, timeout (recv only)
- **Outputs/Return:** Status code; actual bytes sent/received via output parameter
- **Notes:** Timeout in seconds for recv; platform implementations differ in blocking behavior

### LNat_Os_Socket_Connect / Close / Send / Recv (TCP)
- **Signature:** Connect: `int LNat_Os_Socket_Connect(OsSocket **, host_addr, port, timeout_sec)`; Close: `int LNat_Os_Socket_Close(OsSocket **)`; Send/Recv: similar to UDP variants
- **Purpose:** Establish TCP connections, terminate sockets, send/receive stream data
- **Inputs:** Socket handle(s), destination address/port, buffer, byte counts, timeouts
- **Outputs/Return:** Status codes; data transferred via buffers and count output parameters

### LNat_Os_Get_Local_Ip
- **Signature:** `int LNat_Os_Get_Local_Ip(OsSocket *, char ** local_ip)`
- **Purpose:** Query the local IP address bound to a connected socket
- **Inputs:** Socket handle
- **Outputs/Return:** Status code; local IP string via output parameter
- **Calls:** Routed to platform implementation

## Control Flow Notes
This header does not define control flow; it is a compile-time abstraction. At build time, preprocessor conditionals set `OS_UNIX` or `OS_WIN`, and corresponding macros redirect calls to platform-specific function implementations. Actual socket lifecycle (init ΓåÆ setup ΓåÆ send/recv ΓåÆ close) is managed by callers in higher-level LibNAT code.

## External Dependencies
- **Includes:** None (pure C declarations)
- **Preprocessor symbols used:** `_WIN32`, `WIN32`, `__CYGWIN__`, `__MINGW32__`, `__BORLANDC__` (platform detection)
- **Standard C types:** `int`, `char`, `short int` (built-in)

---

**Notes:**
- All socket operation functions are macros that conditionally alias to `LNat_Unix_*` or `LNat_Win_*` implementations.
- `OsSocket` is forward-declared as opaque; its layout is hidden from callers and defined in platform-specific source files.
- Function prototypes declare return codes as `int`; actual error semantics likely documented in companion implementation files.
