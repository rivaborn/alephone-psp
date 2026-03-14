# Source_Files/Network/SSLP_limited.cpp

## File Purpose
Implements the Simple Service Location Protocol (SSLP) for network service discovery in Aleph One. Provides single-threaded, pump-based service discovery and advertisement using SDL_net UDP broadcast, replacing AppleTalk NBP for IP networks. Built as a simplified, non-threaded alternative to support dynamic player location in network games.

## Core Responsibilities
- Service instance discovery: locate and track remote service instances via broadcast FIND/HAVE messages
- Service advertisement: allow local services to be discovered by responding to FIND queries
- Instance lifecycle management: maintain linked list of discovered services with timeout-based expiration
- Network message handling: receive and dispatch FIND, HAVE, and LOST packet types with validation
- Cross-subnet hinting: unicast HAVE announcements to specific hosts in isolated broadcast domains
- Packet serialization: pack/unpack protocol messages with endian-neutral binary format
- Callback invocation: trigger application callbacks for discovered/lost/renamed instances from main thread

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLPint_FoundInstance` | struct | Wraps discovered service instance with timestamp for timeout tracking; forms linked list |
| `SSLP_ServiceInstance` | struct (from API) | Service identity: type (32 chars), name (32 chars), UDP address (host:port) |
| `SSLP_Packet` | struct (from Protocol) | Network packet: magic, version, message type, service type/name/port fields |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sBehaviorsDesired` | int (bitflags) | static | Tracks active subsystems: SSLPINT_LOCATING (searching), SSLPINT_RESPONDING (advertising), SSLPINT_HINTING (unicasting) |
| `sSocketDescriptor` | UDPsocket | static | Shared UDP socket for all SSLP send/receive operations |
| `sReceivingPacket` | UDPpacket* | static | Allocated buffer for each incoming packet |
| `sFindPacket` | UDPpacket* | static | Pre-built broadcast FIND packet (reused each pump cycle) |
| `sFoundInstances` | SSLPint_FoundInstance* | static | Head of linked list of discovered service instances |
| `sFoundCallback`, `sLostCallback`, `sNameChangedCallback` | function pointers | static | Application callbacks invoked on instance events |
| `sResponsePacket` | UDPpacket* | static | Pre-built HAVE response packet sent when responding to FIND |
| `sHintPacket` | UDPpacket* | static | Pre-built HAVE hint packet sent unicast to specific addresses |

## Key Functions / Methods

### SSLPint_FoundAnInstance
- Signature: `static struct SSLP_ServiceInstance* SSLPint_FoundAnInstance(struct SSLP_ServiceInstance* inInstance)`
- Purpose: Register newly discovered service or update existing one; manage sFoundInstances list
- Inputs: `inInstance` ΓÇö service details from received HAVE message
- Outputs/Return: Pointer to internal copy if new instance (caller must not free); NULL if already known
- Side effects: Appends new SSLPint_FoundInstance to linked list, updates timestamp, invokes sNameChangedCallback if name differs
- Calls: malloc, strncmp, strncpy, SDL_GetTicks, sNameChangedCallback
- Notes: Matches by host:port pair (ignores service_type per current limitation). Returned pointer valid until SSLPint_LostAnInstance or SSLPint_RemoveTimedOutInstances removes it.

### SSLPint_RemoveTimedOutInstances
- Signature: `void SSLPint_RemoveTimedOutInstances()`
- Purpose: Expire discovered instances not seen for SSLPINT_INSTANCE_TIMEOUT (20000 ms)
- Inputs: None (reads globals)
- Outputs/Return: None
- Side effects: Removes stale nodes from sFoundInstances list, frees both wrapper and ServiceInstance, calls sLostCallback if SSLPINT_LOCATING
- Calls: SDL_GetTicks, sLostCallback, free
- Notes: Properly handles linked-list removal (tracks previous node). Only triggers lost callbacks when actively locating.

### SSLPint_LostAnInstance
- Signature: `static struct SSLP_ServiceInstance* SSLPint_LostAnInstance(struct SSLP_ServiceInstance* inInstance)`
- Purpose: Unregister a discovered instance (on explicit LOST message or manual withdrawal)
- Inputs: `inInstance` ΓÇö service to match and remove
- Outputs/Return: Pointer to removed ServiceInstance (caller must free); NULL if not found
- Side effects: Removes wrapper from sFoundInstances, frees wrapper only (instance pointer returned for caller cleanup)
- Calls: free
- Notes: Caller responsible for freeing returned ServiceInstance.

### SSLPint_ReceivedPacket
- Signature: `static void SSLPint_ReceivedPacket()`
- Purpose: Dispatch incoming SSLP packet to appropriate handler (FIND/HAVE/LOST)
- Inputs: None (reads sReceivingPacket global)
- Outputs/Return: None
- Side effects: May call SSLPint_FoundAnInstance, SSLPint_LostAnInstance, or send response/broadcast; invokes callbacks
- Calls: UnpackPacket, strncmp, SDLNet_UDP_Send, SDLNetx_UDP_Broadcast, SSLPint_FoundAnInstance, SSLPint_LostAnInstance
- Notes: Validates magic (SSLPP_MAGIC) and version (SSLPP_VERSION) before dispatching. FIND triggers response if SSLPINT_RESPONDING. HAVE/LOST check service_type match.

### SSLPint_Enter
- Signature: `static bool SSLPint_Enter()`
- Purpose: Initialize shared socket and packet buffers on first SSLP activation
- Inputs: None
- Outputs/Return: true on success; false on failure
- Side effects: Opens UDP socket (tries SSLP_PORT, falls back to 0), enables broadcast, allocates sReceivingPacket
- Calls: SDLNet_UDP_Open, SDLNetx_EnableBroadcast, SDLNet_AllocPacket
- Notes: Called implicitly by SSLP_Locate_Service_Instances or SSLP_Allow_Service_Discovery. Asserts sBehaviorsDesired == SSLPINT_NONE on entry.

### SSLPint_Exit
- Signature: `static int SSLPint_Exit()`
- Purpose: Tear down shared socket and packet buffers when all behaviors done
- Inputs: None
- Outputs/Return: Always 0
- Side effects: Closes socket, frees receive packet, nullifies globals
- Calls: SDLNet_UDP_Close, SDLNet_FreePacket
- Notes: Called implicitly by stop/disallow functions when sBehaviorsDesired ΓåÆ SSLPINT_NONE. Asserts sBehaviorsDesired == SSLPINT_NONE.

### SSLP_Locate_Service_Instances
- Signature: `void SSLP_Locate_Service_Instances(const char* inServiceType, SSLP_Service_Instance_Status_Changed_Callback inFoundCallback, SSLP_Service_Instance_Status_Changed_Callback inLostCallback, SSLP_Service_Instance_Status_Changed_Callback inNameChangedCallback)`
- Purpose: Start searching for services of a given type; register discovery callbacks
- Inputs: Service type string; three callback pointers (any may be NULL)
- Outputs/Return: None
- Side effects: Allocates sFindPacket, sets up FIND message, enables SSLPINT_LOCATING, calls SSLPint_Enter if needed, stores callbacks
- Calls: SSLPint_Enter, SDLNet_AllocPacket, PackPacket, strncpy, memset
- Notes: API asserts only one locate attempt active at a time. sFindPacket->data packed on every call.

### SSLP_Stop_Locating_Service_Instances
- Signature: `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- Purpose: Stop service location search and clear all discovered instances
- Inputs: `inServiceType` ΓÇö ignored (current implementation only tracks one)
- Outputs/Return: None
- Side effects: Disables SSLPINT_LOCATING, frees sFindPacket, clears callbacks, flushes sFoundInstances, calls SSLPint_Exit if no other behaviors
- Calls: SDLNet_FreePacket, SSLPint_FlushAllFoundInstances, SSLPint_Exit
- Notes: All previously-returned ServiceInstance pointers become invalid. API asserts SSLPINT_LOCATING is active.

### SSLP_Allow_Service_Discovery
- Signature: `void SSLP_Allow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- Purpose: Advertise a local service instance; respond to incoming FIND queries
- Inputs: Service instance to advertise (type, name, port; host ignored)
- Outputs/Return: None
- Side effects: Allocates sResponsePacket, sets up HAVE message, broadcasts initial HAVE, enables SSLPINT_RESPONDING, calls SSLPint_Enter if needed
- Calls: SSLPint_Enter, SDLNet_AllocPacket, PackPacket, strncpy, SDLNetx_UDP_Broadcast
- Notes: API asserts no prior responding/hinting. Only one service can be advertised at a time.

### SSLP_Hint_Service_Discovery
- Signature: `void SSLP_Hint_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance, const IPaddress* inAddress)`
- Purpose: Periodically unicast HAVE messages to a specific remote address (bridges broadcast-isolated subnets)
- Inputs: Service instance to hint; remote address (in network byte order)
- Outputs/Return: None
- Side effects: Allocates sHintPacket if needed, sets up unicast HAVE, enables SSLPINT_HINTING, auto-enables SSLP_Allow_Service_Discovery if not active
- Calls: SSLP_Allow_Service_Discovery, SDLNet_AllocPacket, PackPacket, strncpy
- Notes: Multiple hints can be active (last one overwrites sHintPacket state; API comment warns this is odd design).

### SSLP_Disallow_Service_Discovery
- Signature: `void SSLP_Disallow_Service_Discovery(const SSLP_ServiceInstance* inInstance)`
- Purpose: Stop advertising service; withdraw from network (send courteous LOST)
- Inputs: `inInstance` ΓÇö ignored (current implementation stops the one advertised service)
- Outputs/Return: None
- Side effects: Stops hinting if active (sends unicast LOST), sends broadcast LOST, disables SSLPINT_RESPONDING, frees packets, calls SSLPint_Exit if no other behaviors
- Calls: UnpackPacket, PackPacket, SDLNet_UDP_Send, SDLNetx_UDP_Broadcast, SDLNet_FreePacket, SSLPint_Exit
- Notes: API asserts SSLPINT_RESPONDING active. LOST messages courtesy only (not required for correct operation).

### SSLP_Pump
- Signature: `void SSLP_Pump()`
- Purpose: Main event loop entry point; called periodically from application to do background work
- Inputs: None (reads globals)
- Outputs/Return: None
- Side effects: Broadcasts FIND packets (every 5 sec), sends hints (every 5 sec), receives and processes all pending packets, removes expired instances
- Calls: SDL_GetTicks, SDLNetx_UDP_Broadcast, SSLPint_RemoveTimedOutInstances, SDLNet_UDP_Send, SDLNet_UDP_Recv, SSLPint_ReceivedPacket
- Notes: Throttled broadcast/hint work to 5-second intervals to avoid network spam. Receiving happens every call. No-op if sBehaviorsDesired == SSLPINT_NONE. **Critical**: call frequently (ideally every frame) for responsiveness.

### PackPacket / UnpackPacket
- Signature: `void PackPacket(unsigned char *Packed, SSLP_Packet *Unpacked)` and inverse
- Purpose: Serialize/deserialize SSLP_Packet between host struct and network byte-order buffer
- Inputs: Source and destination pointers (direction swapped between functions)
- Outputs/Return: None (modifies in-place)
- Side effects: Endian conversion via memcpy (portable across platforms), pointer incremented
- Calls: PacketCopyIn/Out templates, memcpy
- Notes: Templates handle both scalar fields (magic, version, message, ports) and fixed-length arrays (service_type, service_name). Ensures struct packing doesn't vary by platform.

## Control Flow Notes

**Non-threaded, pump-based architecture:**

1. **Setup**: Application calls `SSLP_Locate_Service_Instances()` and/or `SSLP_Allow_Service_Discovery()`, which auto-initialize SSLP on first call via `SSLPint_Enter()`.

2. **Frame loop**: Application calls `SSLP_Pump()` periodically (once per frame recommended; minimum every 5 seconds):
   - Receives all pending packets from socket ΓåÆ `SSLPint_ReceivedPacket()` for each
   - Every 5 seconds: broadcasts queued FIND and hint packets, removes timed-out instances
   - Callbacks invoked in caller's thread (no synchronization needed)

3. **Shutdown**: Application calls `SSLP_Stop_Locating_Service_Instances()` and/or `SSLP_Disallow_Service_Discovery()`, which auto-cleanup via `SSLPint_Exit()` when all behaviors cease.

No distinct render or update phase; all work is network I/O pumped from main loop.

## External Dependencies
- **SDL**: `SDL_GetTicks()`, `SDL_SwapBE16()`, `SDL_SwapBE32()` (endian swap), `SDL_thread.h` (included but not usedΓÇölegacy)
- **SDL_net**: `SDLNet_UDP_Open()`, `SDLNet_UDP_Close()`, `SDLNet_AllocPacket()`, `SDLNet_FreePacket()`, `SDLNet_UDP_Send()`, `SDLNet_UDP_Recv()`
- **SDL_netx** (broadcast extensions): `SDLNetx_EnableBroadcast()`, `SDLNetx_UDP_Broadcast()`
- **C stdlib**: `malloc()`, `free()`, `memcpy()`, `memset()`, `strncpy()`, `strncmp()`
- **Aleph One Logging**: `logTrace()`, `logTrace4()`, `logNote()`, `logNote1()`, `logNote2()`, `logAnomaly()`, `logContext()`, `Logger` class
