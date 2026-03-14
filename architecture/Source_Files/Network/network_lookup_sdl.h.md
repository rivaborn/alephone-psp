# Source_Files/Network/network_lookup_sdl.h

## File Purpose
SDL-based header providing network service discovery and registration functions for multiplayer games. Acts as a wrapper around the SSLP (Simple Service Location Protocol) to enable game instances to register themselves and discover other players on local networks, abstracting platform-specific network discovery (replacing MacOS AppleTalk NBP).

## Core Responsibilities
- Register local game service instances for network discovery
- Unregister service instances when shutting down
- Initiate discovery of remote game service instances via callbacks
- Manage service lookup lifecycle (open/close operations)
- Bridge Aleph One's network interface with SDL-based SSLP implementation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLP_Service_Instance_Status_Changed_Callback` | typedef (function pointer) | Callback signature for service discovery events (found/lost/name-changed) |
| `SSLP_ServiceInstance` | struct (from SSLP_API.h) | Represents a discoverable service with type, name, and IP/port address |

## Global / File-Static State
None.

## Key Functions / Methods

### NetRegisterName
- **Signature:** `OSErr NetRegisterName(const unsigned char *name, const unsigned char *type, short version, short socketNumber, const char* hint_addr_string)`
- **Purpose:** Register a local game service instance for discovery by other players.
- **Inputs:** Service name (Pstring), service type, version identifier, listen socket, optional hint address for cross-network broadcasting.
- **Outputs/Return:** `OSErr` status code (0 = success).
- **Side effects:** Publishes service to local network via SSLP; may broadcast hints to remote hosts.
- **Calls:** Implementation in NETWORK_NAMES.C (not visible here).
- **Notes:** Supports SSLP hinting to reach machines in different broadcast domains. Only one service can be published at a time (SSLP limitation).

### NetUnRegisterName
- **Signature:** `OSErr NetUnRegisterName(void)`
- **Purpose:** Unregister the local service instance from network discovery.
- **Inputs:** None.
- **Outputs/Return:** `OSErr` status code.
- **Side effects:** Stops broadcasting service presence; halts hint broadcasts.
- **Calls:** Implementation in NETWORK_NAMES.C.
- **Notes:** Should be called during shutdown or when game becomes unavailable.

### NetLookupOpen_SSLP
- **Signature:** `OSErr NetLookupOpen_SSLP(const unsigned char *type, short version, SSLP_Service_Instance_Status_Changed_Callback foundInstance, SSLP_Service_Instance_Status_Changed_Callback lostInstance, SSLP_Service_Instance_Status_Changed_Callback nameChanged)`
- **Purpose:** Begin active discovery of remote game service instances of a given type.
- **Inputs:** Service type to search for (Pstring), version filter, three optional callback functions.
- **Outputs/Return:** `OSErr` status code.
- **Side effects:** Initiates asynchronous SSLP broadcast queries; callbacks may fire from different threads as instances are discovered or lost.
- **Calls:** Implementation in NETWORK_LOOKUP.C; delegates to underlying SSLP.
- **Notes:** Only one concurrent search allowed. Callbacks are invoked per event (foundInstance, lostInstance, nameChanged). Requires periodic `SSLP_Pump()` calls in non-threaded implementations.

### NetLookupClose
- **Signature:** `void NetLookupClose(void)`
- **Purpose:** Stop active service discovery.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Ceases service lookups; invalidates all discovered service pointers.
- **Calls:** Implementation in NETWORK_LOOKUP.C.
- **Notes:** Called when player cancels server browser or game transitions.

## Control Flow Notes
This file appears in the **game startup/menu phase**: when a player initiates multiplayer, `NetLookupOpen_SSLP()` is called to populate server lists; when hosting, `NetRegisterName()` publishes the game. Shutdown calls `NetLookupClose()` and `NetUnRegisterName()`. Relies on external `SSLP_Pump()` calls (likely from a UI dialog event loop or frame update) to drive asynchronous discovery.

## External Dependencies
- **SSLP_API.h** ΓÇô Provides `SSLP_ServiceInstance`, callback typedef, and low-level SSLP functions (`SSLP_Pump()`, `SSLP_Locate_Service_Instances()`, `SSLP_Allow_Service_Discovery()`, etc.).
- **SDL_net** (via SSLP_API.h) ΓÇô Cross-platform UDP/IP networking.
- **NETWORK_NAMES.C** ΓÇô Implementation of registration functions.
- **NETWORK_LOOKUP.C** ΓÇô Implementation of discovery/lookup functions.
- **Aleph One core** ΓÇô Defines `OSErr` and Pstring conventions.
