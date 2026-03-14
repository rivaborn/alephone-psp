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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `UpnpController` | struct | Encapsulates control URL and service type for communicating with a discovered UPnP IGD |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Upnp_Discover
- **Signature:** `int LNat_Upnp_Discover(UpnpController ** c)`
- **Purpose:** Sends SSDP discovery requests to locate a UPnP-enabled IGD providing WANIPConnection or WANPPPConnection services
- **Inputs:** Pointer to `UpnpController` pointer (output parameter)
- **Outputs/Return:** Returns 0 on success; populates `c` with newly allocated controller; non-zero error code on failure
- **Side effects:** Allocates memory for `UpnpController`; sends network discovery multicast; caller must later free via `LNat_Upnp_Controller_Free`
- **Calls:** Not inferable from this file
- **Notes:** Caller responsible for deallocation; returns immediately if no IGD found (via error code)

### LNat_Upnp_Get_Public_Ip
- **Signature:** `int LNat_Upnp_Get_Public_Ip(const UpnpController * controller, char * public_ip, int public_ip_size)`
- **Purpose:** Retrieves the external/public IP address of the IGD from the local network
- **Inputs:** Discovered controller; pre-allocated buffer `public_ip`; buffer size
- **Outputs/Return:** Returns 0 on success; writes null-terminated IP string to `public_ip`; truncates if buffer insufficient
- **Side effects:** Modifies `public_ip` buffer; performs SOAP request to IGD
- **Calls:** Not inferable from this file
- **Notes:** Buffer size must be sufficient; result always null-terminated

### LNat_Upnp_Set_Port_Mapping
- **Signature:** `int LNat_Upnp_Set_Port_Mapping(const UpnpController * controller, const char * ip_map, short int port_map, const char* protocol)`
- **Purpose:** Creates a port mapping in the IGD to forward inbound traffic to this machine's local port
- **Inputs:** Controller; local IP (or NULL to auto-detect); port number; protocol string ("TCP" or "UDP")
- **Outputs/Return:** Returns 0 on success; non-zero error code on failure
- **Side effects:** Modifies IGD port mapping table; enables forwarding; performs SOAP request
- **Calls:** Not inferable from this file
- **Notes:** If `ip_map` is NULL, function attempts to determine local IP automatically

### LNat_Upnp_Remove_Port_Mapping
- **Signature:** `int LNat_Upnp_Remove_Port_Mapping(const UpnpController * controller, short int port_map, const char* protocol)`
- **Purpose:** Deletes a port mapping from the IGD to revoke port forwarding
- **Inputs:** Controller; port number; protocol string ("TCP" or "UDP")
- **Outputs/Return:** Returns 0 on success; non-zero error code on failure
- **Side effects:** Modifies IGD port mapping table; disables forwarding; frees port for other clients
- **Calls:** Not inferable from this file
- **Notes:** Should be called before application shutdown to clean up

### LNat_Upnp_Controller_Free
- **Signature:** `int LNat_Upnp_Controller_Free(UpnpController ** controller)`
- **Purpose:** Deallocates resources allocated by `LNat_Upnp_Discover`
- **Inputs:** Pointer to controller to free
- **Outputs/Return:** Returns 0 on success
- **Side effects:** Frees dynamically allocated memory (control URL, service type); invalidates controller
- **Calls:** Not inferable from this file
- **Notes:** Must be called to prevent memory leaks; no error recovery possible if pointer invalid

### LNat_Print_Error
- **Signature:** `void LNat_Print_Error(int error)`
- **Purpose:** Utility function to print human-readable error descriptions to stderr
- **Inputs:** Error code (typically return value from other functions)
- **Outputs/Return:** void
- **Side effects:** Writes error message to stderr
- **Calls:** Not inferable from this file

## Control Flow Notes
This is a header file; no executable flow is present. Typical caller usage is: discover IGD ΓåÆ query public IP ΓåÆ set port mappings ΓåÆ (later) remove mappings ΓåÆ free controller.

## External Dependencies
- `error.h` ΓÇö defines error code constants (`OK`, `BAD_MALLOC`, `UPNP_*`, `SOCKET_*`, `HTTP_*`, `SSDP_*`, etc.) used as return values
- Platform headers (Windows): conditional `__declspec(dllexport/dllimport)` for DLL build support (currently disabled)
- Standard C library types: `char`, `int`, `short int`
