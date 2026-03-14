# Source_Files/Network/network_lookup_sdl.cpp

## File Purpose
SDL-based implementation of network service discovery and registration for Aleph One using the SSLP (Simple Service Location Protocol). Bridges the game's networking layer with cross-platform SSLP for discovering available game servers and advertising the local player to other clients.

## Core Responsibilities
- Build SSLP-compatible service type strings (combining service type + version)
- Start/stop searching for available network services via callbacks
- Register and unregister the local player/service instance for discovery
- Maintain lookup and registration state (two file-static bools)
- Support address hinting for inter-domain service discovery
- Convert between Pascal strings (P-strings) and C-style null-terminated strings

## Key Types / Data Structures
None (uses external types only: `SSLP_ServiceInstance`, `NetAddrBlock`, `NetEntityName`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sNameRegistered` | bool | static | Tracks if local service is currently advertised |
| `sLookupInProgress` | bool | static | Tracks if service discovery is currently active |

## Key Functions / Methods

### NetLookup_BuildTypeString
- Signature: `static void NetLookup_BuildTypeString(char* outDest, const unsigned char* inType, short inVersion)`
- Purpose: Convert P-string service type + version number into a C-string-compatible format
- Inputs: destination buffer, P-string type, version short
- Outputs/Return: outDest modified in-place (P-string format)
- Side effects: writes to buffer; truncates type to `SSLP_MAX_TYPE_LENGTH - 8` to reserve space for length byte, null terminator, and version string
- Calls: `memcpy`, `sprintf`, `strlen`
- Notes: Result is still P-string format (length byte first); null-terminated for C compatibility

### NetLookupOpen_SSLP
- Signature: `OSErr NetLookupOpen_SSLP(const unsigned char *type, short version, SSLP_Service_Instance_Status_Changed_Callback foundInstance, SSLP_Service_Instance_Status_Changed_Callback lostInstance, SSLP_Service_Instance_Status_Changed_Callback nameChanged)`
- Purpose: Initiate search for services of a given type/version; callbacks invoked as instances appear/disappear/rename
- Inputs: service type (P-string), version, three callback function pointers
- Outputs/Return: `OSErr` (0 = success)
- Side effects: sets `sLookupInProgress = true`; calls SSLP API to start listening
- Calls: `NetLookup_BuildTypeString`, `SSLP_Locate_Service_Instances`
- Notes: Only one lookup allowed at a time (SSLP API limit); callbacks fire asynchronously

### NetLookupClose
- Signature: `void NetLookupClose(void)`
- Purpose: Stop searching for services; cease invoking callbacks
- Outputs/Return: void
- Side effects: sets `sLookupInProgress = false` if lookup was active
- Calls: `SSLP_Stop_Locating_Service_Instances`
- Notes: Guarded by state check; no-op if not searching

### NetRegisterName
- Signature: `OSErr NetRegisterName(const unsigned char *name, const unsigned char *type, short version, short socketNumber, const char* hint_addr_string)`
- Purpose: Register and advertise the local player/service instance for remote discovery
- Inputs: player name (P-string), service type (P-string), version, socket port, optional hint address string
- Outputs/Return: `OSErr` (0 = success; error code if address hint resolution fails)
- Side effects: 
  - sets `sNameRegistered = true`
  - publishes service via SSLP
  - if hint provided: resolves address and calls SSLP hinting API
- Calls: `NetLookup_BuildTypeString`, `SSLP_Allow_Service_Discovery`, `SDLNet_ResolveHost`, `SSLP_Hint_Service_Discovery`, `strdup`, `free`
- Notes: 
  - Truncates name to `SSLP_MAX_NAME_LENGTH - 2`
  - Makes temporary copy of hint string because SDL_net lacks const-correctness
  - Only port is used; host part is ignored by SSLP
  - Only one service registered at a time (SSLP API limit)

### NetUnRegisterName
- Signature: `OSErr NetUnRegisterName(void)`
- Purpose: Stop advertising the local player/service instance
- Outputs/Return: `OSErr` (0)
- Side effects: sets `sNameRegistered = false`; stops hinting if active
- Calls: `SSLP_Disallow_Service_Discovery`
- Notes: Guarded by state check; passing NULL to SSLP disables all published services

**Other functions** (stubs): `NetLookupRemove`, `NetLookupInformation`, `NetLookupUpdate` ΓÇö not yet implemented.

## Control Flow Notes
Acts as **service discovery orchestrator** for multiplayer:
1. Game calls `NetLookupOpen_SSLP` ΓåÆ SSLP begins finding servers (callbacks fire as services found/lost)
2. Simultaneously, game calls `NetRegisterName` ΓåÆ local player advertised for remote discovery
3. On shutdown: `NetUnRegisterName` + `NetLookupClose` clean up state

Single-threaded SSLP implementation implies callbacks execute during `SSLP_Pump()` calls (not shown here); threading and pump invocation handled elsewhere.

Wrapped in `#if !defined(DISABLE_NETWORKING)` conditional compilation.

## External Dependencies
- `SSLP_API.h`: `SSLP_ServiceInstance`, `SSLP_Locate_Service_Instances`, `SSLP_Stop_Locating_Service_Instances`, `SSLP_Allow_Service_Discovery`, `SSLP_Hint_Service_Discovery`, `SSLP_Disallow_Service_Discovery`, constants `SSLP_MAX_TYPE_LENGTH`, `SSLP_MAX_NAME_LENGTH`
- `SDL_net`: `IPaddress`, `SDLNet_ResolveHost`
- `sdl_network.h`: `NetAddrBlock`, `NetEntityName`, `OSErr`
- Standard C: `memcpy`, `sprintf`, `strlen`, `strdup`, `free`
