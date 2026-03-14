# Source_Files/Network/ConnectPool.cpp

## File Purpose

Implements a non-blocking TCP connection pool for the Aleph One engine. `NonblockingConnect` wraps individual asynchronous socket connections in SDL threads, while `ConnectPool` is a singleton that manages a fixed-size pool of these connections, allocating on demand and cleaning up finished connections.

## Core Responsibilities

- **NonblockingConnect**: Establish a single TCP connection asynchronously in a dedicated thread; separate DNS resolution from connection to avoid blocking.
- **ConnectPool**: Singleton pool managementΓÇöallocate connection slots on demand, track in-use vs. available slots, and perform lazy cleanup of completed connections.
- **Thread management**: Spawn, track, and join SDL threads for each connection attempt; propagate connection status back to caller.
- **Memory safety**: Use `std::auto_ptr` to ensure cleanup of transient buffers and channels; support explicit abandonment by caller.

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect` | class | Encapsulates a single non-blocking TCP connection, with DNS resolution and status tracking. |
| `ConnectPool` | class | Singleton pool holding up to 20 `NonblockingConnect` objects with in-use/available slots. |
| `Status` enum | enum | Connection lifecycle states: `Connecting`, `Connected`, `ResolutionFailed`, `ConnectFailed`. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ConnectPool::m_instance` | `ConnectPool*` | static | Singleton instance pointer; initialized to null. |

## Key Functions / Methods

### NonblockingConnect::NonblockingConnect(const std::string&, uint16)
- **Signature:** `NonblockingConnect(const std::string& address, uint16 port)`
- **Purpose:** Construct a connection object with hostname and port; immediately spawn a background thread to resolve and connect.
- **Inputs:** `address` (hostname/IP as string), `port` (TCP port).
- **Outputs/Return:** Object initialized; `m_thread` spawned.
- **Side effects:** Allocates SDL thread; sets `m_status = Connecting`.
- **Calls:** `connect()`.
- **Notes:** Does not block; caller must poll `done()` or check `status()`.

### NonblockingConnect::NonblockingConnect(const IPaddress&)
- **Signature:** `NonblockingConnect(const IPaddress& ip)`
- **Purpose:** Construct a connection object from a pre-resolved IP address; skips DNS resolution.
- **Inputs:** `ip` (resolved socket address).
- **Outputs/Return:** Object initialized; `m_thread` spawned.
- **Side effects:** Allocates SDL thread; sets `m_ipSpecified = true`, `m_status = Connecting`.
- **Calls:** `connect()`.

### NonblockingConnect::Thread()
- **Signature:** `int Thread()`
- **Purpose:** Main work function executed in background thread; resolve DNS (if needed) and establish socket connection.
- **Inputs:** None (uses member state: `m_address`, `m_port`, `m_ip`, `m_ipSpecified`).
- **Outputs/Return:** Integer status code (0 on success, 1 on resolution failure, 2 on connection failure).
- **Side effects:** Updates `m_status`; allocates and stores `CommunicationsChannel`; may call `SDLNet_ResolveHost`.
- **Calls:** `SDLNet_ResolveHost`, `channel->connect()`, `channel->isConnected()`.
- **Notes:** Runs in separate thread to avoid blocking. Resolution and connection errors set `m_status` to `ResolutionFailed` or `ConnectFailed`.

### NonblockingConnect::connect_thread (static)
- **Signature:** `static int connect_thread(void *p)`
- **Purpose:** SDL thread entry point; delegates to instance's `Thread()` method.
- **Inputs:** `p` (opaque pointer cast to `NonblockingConnect*`).
- **Outputs/Return:** Delegates return value from `Thread()`.
- **Calls:** `Thread()`.

### ConnectPool::connect(const std::string&, uint16)
- **Signature:** `NonblockingConnect* connect(const std::string& address, uint16 port)`
- **Purpose:** Allocate a connection slot from the pool for the given address and port.
- **Inputs:** `address` (hostname), `port` (TCP port).
- **Outputs/Return:** Pointer to newly allocated `NonblockingConnect`, or null if pool exhausted.
- **Side effects:** Calls `fast_free()`; marks slot as in-use by setting second=false.
- **Calls:** `fast_free()`, `NonblockingConnect` constructor.
- **Notes:** Returns immediately; connection happens asynchronously. Caller should check `done()` periodically.

### ConnectPool::connect(const IPaddress&)
- **Signature:** `NonblockingConnect* connect(const IPaddress& ip)`
- **Purpose:** Allocate a connection slot for a pre-resolved IP address (faster path, skips DNS).
- **Inputs:** `ip` (resolved socket address).
- **Outputs/Return:** Pointer to newly allocated `NonblockingConnect`, or null if pool exhausted.
- **Side effects:** Calls `fast_free()`; marks slot as in-use.
- **Calls:** `fast_free()`, `NonblockingConnect` constructor.

### ConnectPool::fast_free()
- **Signature:** `void fast_free()`
- **Purpose:** Scan pool and delete completed `NonblockingConnect` objects.
- **Inputs:** None (uses `m_pool`).
- **Outputs/Return:** None.
- **Side effects:** Deletes finished connections; nullifies pool slots; **does not reset second=true** (apparent bug?).
- **Calls:** `NonblockingConnect::done()`, `delete`.
- **Notes:** Called at start of each `connect()` to recycle stale slots. Note that marked slots (`second==false`) are never freed hereΓÇöonly already-abandoned slots (`second==true`) with finished connections are cleaned.

### ConnectPool::abandon(NonblockingConnect*)
- **Signature:** `void abandon(NonblockingConnect* nbc)`
- **Purpose:** Mark a connection as available for cleanup and reuse.
- **Inputs:** `nbc` (pointer to a `NonblockingConnect` from the pool).
- **Outputs/Return:** None.
- **Side effects:** Sets `second=true` for the slot, allowing `fast_free()` to delete it later.
- **Calls:** None.

## Control Flow Notes

This is asynchronous network code that operates outside the main game frame loop. Typical usage flow:

1. Game code calls `ConnectPool::instance()->connect(address, port)` to initiate a connection.
2. The pool spawns a background thread and returns immediately.
3. Game code polls `NonblockingConnect::done()` or checks `status()` each frame.
4. When `done()` returns true and `status() == Connected`, game code calls `release()` to obtain the `CommunicationsChannel`.
5. Game code calls `abandon(nbc)` to return the slot to the pool for cleanup.
6. Next call to `connect()` may reuse the freed slot after `fast_free()` deletes the old object.

## External Dependencies

- **SDL threading:** `SDL_CreateThread`, `SDL_WaitThread` (from `SDL_thread.h`).
- **SDL networking:** `SDLNet_ResolveHost` (from SDL_net library, not explicitly included here but used).
- **CommunicationsChannel:** Defined elsewhere; wraps actual socket I/O.
- **cseries.h:** Likely defines `uint16`, `IPaddress`, and assert macros.
- **Standard library:** `<string>`, `<memory>` (for `std::auto_ptr`).

---

### Potential Issues

- **Memory leak in `fast_free()`:** Slots marked `second=true` are deleted, but the boolean is never reset to true, preventing slot reuse (though they are removed from the list).
- **Destructor disabled:** Cleanup code is commented out, preventing proper resource release at exit.
- **std::auto_ptr deprecated:** This code uses C++98 idiom; modern code would use `std::unique_ptr`.
