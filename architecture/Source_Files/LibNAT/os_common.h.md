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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OsSocket | struct | Opaque socket wrapper; contains `int sock` (Unix) or `SOCKET sock` (Windows) |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Common_Socket_Udp_Send
- Signature: `int LNat_Common_Socket_Udp_Send(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_sent)`
- Purpose: Send data over a UDP socket to a specified host and port
- Inputs: socket pointer, destination host address (string), port, buffer, bytes to send
- Outputs/Return: status code (int), actual bytes sent via `amt_sent` pointer
- Side effects: network I/O
- Calls: (defined elsewhere, OS-specific implementation)
- Notes: connectionless; target address must be specified per send

### LNat_Common_Socket_Udp_Recv
- Signature: `int LNat_Common_Socket_Udp_Recv(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_recv, int timeout_sec)`
- Purpose: Receive data over a UDP socket from a specified host and port
- Inputs: socket pointer, source host address (string), port, receive buffer, buffer size, timeout in seconds
- Outputs/Return: status code (int), actual bytes received via `amt_recv` pointer
- Side effects: network I/O, blocks until timeout or data available
- Calls: (defined elsewhere, OS-specific implementation)
- Notes: timeout_sec controls blocking duration

### LNat_Common_Socket_Send
- Signature: `int LNat_Common_Socket_Send(OsSocket * s, char * buf, int amt, int * amt_sent)`
- Purpose: Send data over a connected TCP socket
- Inputs: socket pointer, buffer, bytes to send
- Outputs/Return: status code (int), actual bytes sent via `amt_sent` pointer
- Side effects: network I/O
- Calls: (defined elsewhere)
- Notes: for connected sockets only; no address needed

### LNat_Common_Socket_Recv
- Signature: `int LNat_Common_Socket_Recv(OsSocket * s, char * buf, int amt, int * amt_recv, int timeout_sec)`
- Purpose: Receive data over a connected TCP socket
- Inputs: socket pointer, receive buffer, buffer size, timeout in seconds
- Outputs/Return: status code (int), actual bytes received via `amt_recv` pointer
- Side effects: network I/O, blocks until timeout or data available
- Calls: (defined elsewhere)
- Notes: timeout_sec controls blocking duration

### Common_Initialize_Sockaddr_in
- Signature: `int Common_Initialize_Sockaddr_in(struct sockaddr_in* server, struct hostent** hp, const char * host_addr, short int port)`
- Purpose: Initialize socket address structure from host and port
- Inputs: pointer to sockaddr_in, pointer to hostent pointer, host address (string), port
- Outputs/Return: status code (int), populated `server` and `hp` structures
- Side effects: DNS lookup (may perform hostname resolution)
- Calls: (defined elsewhere)
- Notes: sets up sockaddr_in for connection or binding

### Select_Till_Readyread
- Signature: `int Select_Till_Readyread(OsSocket * s, int timeout_sec)`
- Purpose: Block until socket is readable or timeout expires
- Inputs: socket pointer, timeout in seconds
- Outputs/Return: status code (int)
- Side effects: blocks via select(); I/O multiplexing primitive
- Calls: select() (system call, not defined here)
- Notes: supports non-blocking I/O patterns

### Select_Till_Readywrite
- Signature: `int Select_Till_Readywrite(OsSocket * s, int timeout_sec)`
- Purpose: Block until socket is writable or timeout expires
- Inputs: socket pointer, timeout in seconds
- Outputs/Return: status code (int)
- Side effects: blocks via select(); I/O multiplexing primitive
- Calls: select() (system call, not defined here)
- Notes: supports non-blocking I/O patterns

### LNat_Common_Get_Local_Ip
- Signature: `int LNat_Common_Get_Local_Ip(OsSocket * s, char ** local_ip)`
- Purpose: Retrieve local IP address bound to a connected socket
- Inputs: socket pointer
- Outputs/Return: status code (int), local IP address string via `local_ip` pointer
- Side effects: queries socket state; may allocate memory for IP string
- Calls: (defined elsewhere)
- Notes: used to determine which local interface is in use

## Control Flow Notes
This header is the **interface contract** for the OS abstraction layer. The actual flow is:
1. Application calls `LNat_Os_*` macros (from `os.h`)
2. Macros expand to OS-specific implementations (`LNat_Unix_*` or `LNat_Win_*`)
3. Implementations use these common declarations and the `OsSocket` structure
4. The opaque `OsSocket` struct hides platform differences from callers

## External Dependencies
- **os.h**: platform detection (`OS_UNIX`, `OS_WIN`) and function interface macros
- **Unix headers** (conditional): fcntl.h, time.h, sys/types.h, sys/socket.h, sys/select.h, netinet/in.h, arpa/inet.h, netdb.h, stdlib.h, unistd.h, sys/time.h, string.h
- **Windows headers** (conditional): windows.h, winsock.h
- **Forward-declared structures**: `struct sockaddr_in` (netinet/in.h or winsock.h), `struct hostent` (netdb.h or winsock.h)
