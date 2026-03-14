# Source_Files/LibNAT/ssdp.h

## File Purpose
Header file for SSDP (Simple Service Discovery Protocol) functionality in a NAT/UPnP library. Declares a single function to send SSDP discovery requests and receive router responses, typically used for UPnP device discovery.

## Core Responsibilities
- Declare SSDP service discovery interface
- Enable sending discovery requests to network devices
- Provide mechanism for receiving discovery responses from routers/UPnP devices

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Ssdp_Discover

- **Signature:** `int LNat_Ssdp_Discover(const char * search_target, char ** response)`
- **Purpose:** Send an SSDP M-SEARCH request and retrieve the response from a discovered device.
- **Inputs:**
  - `search_target` ΓÇô SSDP search target string (e.g., "ssdp:all", device type URN)
  - `response` ΓÇô pointer to char pointer; caller provides address, function fills with response data
- **Outputs/Return:** Integer status/result code (0 likely indicates success, non-zero error).
- **Side effects (global state, I/O, alloc):** Dynamically allocates memory for the response string (caller must free with `free()`). Performs network I/O (UDP multicast to SSDP address).
- **Calls (direct calls visible in this file):** None visible in header.
- **Notes:** Caller must manage response pointer lifecycle. No error detail visible from signature alone; suggests callers should check return value before dereferencing response.

## Control Flow Notes
Likely part of initialization or device discovery phase in a UPnP/NAT traversal stack. Typical use: discover UPnP-capable routers on the local network to enable port mapping and address translation queries.

## External Dependencies
- Assumes standard C library `free()` for caller cleanup
- SSDP protocol implementation (defined elsewhere in LibNAT)
