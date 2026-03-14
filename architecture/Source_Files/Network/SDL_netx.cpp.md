# Source_Files/Network/SDL_netx.cpp

## File Purpose
Extends SDL_net with UDP broadcast capabilities for cross-platform networking. Provides functions to enable broadcast mode on sockets, send packets to all available broadcast addresses, and discover system interface broadcast addresses on non-Windows platforms.

## Core Responsibilities
- Enable/disable SO_BROADCAST socket option on UDP sockets
- Send UDP packets to broadcast addresses (platform-aware implementation)
- Collect and cache broadcast addresses from system interfaces via `ioctl()` (POSIX platforms)
- Handle platform-specific differences (Windows, BeOS, Mac, Linux, PSP, Solaris)
- Manage file-static broadcast address cache to avoid repeated system queries

## Key Types / Data Structures
None (uses SDL_net and standard C socket types; no custom types defined).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sCollectedBroadcastAddresses` | `bool` | static | Flag indicating if broadcast addresses have been queried |
| `sNumberOfBroadcastAddresses` | `int` | static | Count of valid broadcast addresses in cache |
| `sBroadcastAddresses` | `long[8]` | static | Array of cached broadcast addresses (host-part only) |

## Key Functions / Methods

### SDLNetx_EnableBroadcast
- **Signature:** `int SDLNetx_EnableBroadcast(UDPsocket inSocket)`
- **Purpose:** Enable broadcast mode on a UDP socket so SDLNetx_UDP_Broadcast can succeed.
- **Inputs:** `inSocket` ΓÇô UDP socket to enable broadcasting on
- **Outputs/Return:** Positive (non-zero) if broadcast enabled; 0 if failed
- **Side effects:** Triggers SDLNetxint_CollectBroadcastAddresses on non-Windows platforms; calls `setsockopt(SO_BROADCAST)` on underlying socket FD
- **Calls:** `SDLNetxint_CollectBroadcastAddresses()`, `setsockopt()`
- **Notes:** Extracts raw socket FD via unsafe cast `((int*)inSocket)[1]` (depends on SDL_net struct layout); returns 0 on BeOS/Metrowerks (no-op); Windows/Mac usually allow 255.255.255.255 directly

### SDLNetx_DisableBroadcast
- **Signature:** `int SDLNetx_DisableBroadcast(UDPsocket inSocket)`
- **Purpose:** Disable broadcast mode on a UDP socket.
- **Inputs:** `inSocket` ΓÇô UDP socket to disable broadcasting on
- **Outputs/Return:** Positive if broadcast disabled; 0 if failed
- **Side effects:** Calls `setsockopt(SO_BROADCAST, 0)` on underlying socket FD
- **Calls:** `setsockopt()`
- **Notes:** Mirror of Enable; unsafe socket FD extraction; no-op on BeOS/Metrowerks

### SDLNetx_UDP_Broadcast
- **Signature:** `int SDLNetx_UDP_Broadcast(UDPsocket inSocket, UDPpacket* inPacket)`
- **Purpose:** Send `inPacket` to all discovered broadcast addresses.
- **Inputs:** `inSocket` ΓÇô enabled UDP socket; `inPacket` ΓÇô packet to broadcast (port preserved, host overwritten)
- **Outputs/Return:** Number of interfaces the packet was successfully sent on (platform-dependent); updates `inPacket->status` on last send
- **Side effects:** Temporarily modifies `inPacket->address.host` for each send, then restores original
- **Calls:** `SDLNet_UDP_Send()` (from SDL_net)
- **Notes:** POSIX version iterates `sBroadcastAddresses` array; Win32/BeOS/Mac version uses hardcoded 0xffffffff (255.255.255.255)

### SDLNetxint_CollectBroadcastAddresses (static)
- **Signature:** `int SDLNetxint_CollectBroadcastAddresses(UDPsocket inSocket)`
- **Purpose:** Query OS for available network interfaces and extract broadcast addresses (POSIX platforms only; PSP unsupported).
- **Inputs:** `inSocket` ΓÇô UDP socket to use for ioctl queries
- **Outputs/Return:** Number of broadcast addresses collected
- **Side effects:** Populates `sBroadcastAddresses` array and `sNumberOfBroadcastAddresses`; sets `sCollectedBroadcastAddresses = true` regardless of success
- **Calls:** `ioctl(SIOCGIFCONF, SIOCGIFBRDADDR)`
- **Notes:** Uses 1024-byte static buffer; iterates interface list with variable-length entries; filters for AF_INET only; assumes `_SIZEOF_ADDR_IFREQ` macro or falls back to `sizeof(ifreq)`; not compiled on WIN32, BeOS, Mac, __MWERKS__, or PSP

## Control Flow Notes
Not integrated into a game loop; called explicitly by application code at socket initialization time and during game networking phases. No frame-based or render-based dependencies.

## External Dependencies
- **SDL_net**: `UDPsocket`, `UDPpacket` types; `SDLNet_UDP_Send()` function
- **Platform socket APIs**:
  - Windows: `<winsock.h>` (setsockopt, socket types)
  - POSIX: `<sys/socket.h>`, `<netinet/in.h>`, `<sys/ioctl.h>`, `<net/if.h>` (ioctl, ifreq, sockaddr_in)
  - Solaris: BSD_COMP macro for SIOC* definitions
