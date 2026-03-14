# Source_Files/Network/SSLP_API.h

## File Purpose
Public API header for SSLP (Simple Service Location Protocol), a service discovery library for locating and advertising network services across non-AppleTalk networks. Built as a callback-driven alternative to NBP (Name Binding Protocol) for the Aleph One game engine, leveraging SDL_net for cross-platform IP networking.

## Core Responsibilities
- Define the `SSLP_ServiceInstance` data structure for representing discoverable network services
- Provide callback-based APIs for searching for remote services with found/lost/name-changed notifications
- Provide APIs for advertising local services to the network and hinting specific remote hosts
- Supply a pump function for non-threaded implementations requiring periodic processing time
- Define callback function pointer types for service status change events

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLP_ServiceInstance` | struct | Represents a network service with type, name, and network address; returned to callbacks |
| `SSLP_Service_Instance_Status_Changed_Callback` | function pointer typedef | Signature for all three callback types (found, lost, name changed) |

## Global / File-Static State
None.

## Key Functions / Methods

### SSLP_Pump
- **Signature:** `void SSLP_Pump()`
- **Purpose:** Gives the SSLP library processing time for asynchronous operations (non-threaded implementations only).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Processes incoming discovery packets, invokes callbacks for status changes.
- **Calls:** (implementation-dependent, not visible)
- **Notes:** Should be called roughly once per second or more frequently for responsiveness; avoid gaps exceeding 5 seconds. Threaded implementations do not require this call.

### SSLP_Locate_Service_Instances
- **Signature:** `void SSLP_Locate_Service_Instances(const char* inServiceType, SSLP_Service_Instance_Status_Changed_Callback inFoundCallback, SSLP_Service_Instance_Status_Changed_Callback inLostCallback, SSLP_Service_Instance_Status_Changed_Callback inNameChangedCallback)`
- **Purpose:** Begin searching for service instances matching a type on the network.
- **Inputs:** Service type string; three optional callback pointers (found, lost, name-changed).
- **Outputs/Return:** None; results delivered via callbacks.
- **Side effects:** Initiates network discovery; callbacks may execute in different threads.
- **Calls:** (triggers callbacks; implementation-dependent)
- **Notes:** Only one search operation at a time is supported. ServiceInstance pointers remain valid until lost callback or explicit stop. No wildcard/regex supported. Error if called twice without intervening stop call.

### SSLP_Stop_Locating_Service_Instances
- **Signature:** `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- **Purpose:** Stop searching for services of a given type.
- **Inputs:** Service type string (currently ignored; stops all ongoing searches).
- **Outputs/Return:** None.
- **Side effects:** Terminates discovery callbacks immediately; all previously found ServiceInstance pointers become invalid.
- **Calls:** (implementation-dependent)
- **Notes:** No lost callbacks are generated for invalidated instances. Argument parameter is effectively ignored. Error if called without an active search.

### SSLP_Allow_Service_Discovery
- **Signature:** `void SSLP_Allow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- **Purpose:** Advertise a local service instance to the network so remote machines may discover it.
- **Inputs:** Pointer to ServiceInstance struct; address.host is ignored, address.port specifies the local listening port.
- **Outputs/Return:** None.
- **Side effects:** Copies ServiceInstance data; begins broadcasting announcements on the network.
- **Calls:** (implementation-dependent)
- **Notes:** Only one advertised service at a time. Input storage may be freed after call. Error if called twice without intervening disallow call.

### SSLP_Hint_Service_Discovery
- **Signature:** `void SSLP_Hint_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance, const IPaddress* inRemoteHost)`
- **Purpose:** Periodically announce a service to a specific remote machine/port (for cross-subnet discovery).
- **Inputs:** ServiceInstance and remote host address (must be in network byte order); if port is 0, default SSLP_PORT is used.
- **Outputs/Return:** None.
- **Side effects:** Copies provided data; calls Allow_Service_Discovery if not already called; begins targeted unicast announcements.
- **Calls:** Potentially calls `SSLP_Allow_Service_Discovery()` internally.
- **Notes:** ServiceInstance must match the one from Allow_Service_Discovery. Replaces prior hint behavior if called again.

### SSLP_Disallow_Service_Discovery
- **Signature:** `void SSLP_Disallow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- **Purpose:** Stop advertising a local service and cease hinting to remote hosts.
- **Inputs:** ServiceInstance pointer (currently ignored; unpublishes the current service) or NULL.
- **Outputs/Return:** None.
- **Side effects:** Stops network broadcasts; invalidates advertised ServiceInstance pointers on remote machines.
- **Calls:** (implementation-dependent)
- **Notes:** Argument is ignored; only one service at a time supported. Error if called twice without intervening allow call.

## Control Flow Notes
This header defines a **service location loop** for networked game client/server discovery:
- **Service seeker**: Call `SSLP_Locate_Service_Instances()` ΓåÆ periodic `SSLP_Pump()` ΓåÆ receive callbacks ΓåÆ call `SSLP_Stop_Locating_Service_Instances()`
- **Service advertiser**: Call `SSLP_Allow_Service_Discovery()` ΓåÆ optional `SSLP_Hint_Service_Discovery()` for remote hints ΓåÆ periodic `SSLP_Pump()` ΓåÆ call `SSLP_Disallow_Service_Discovery()`
- **Non-threaded mode** requires regular pump calls; **threaded mode** handles itself (comment suggests internal threads manage timing).

## External Dependencies
- **SDL_net.h**: Provides `IPaddress` struct for network addresses.
- **Implementation file** (not shown): Contains state management, UDP broadcast logic, and packet format definitions.
