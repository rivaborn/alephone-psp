# Source_Files/Network/ConnectPool.h

## File Purpose
Provides a singleton connection pool manager for non-blocking outbound TCP connections. Manages up to 20 concurrent asynchronous connection attempts, offloading each attempt to an SDL thread to avoid blocking the main game loop.

## Core Responsibilities
- **NonblockingConnect**: Encapsulates a single asynchronous TCP connection attempt with status tracking and thread lifecycle management
- **ConnectPool (singleton)**: Maintains a reusable pool of up to 20 `NonblockingConnect` objects
- Asynchronous connection initiation via separate SDL threads (non-blocking to caller)
- Status polling to check connection progress (Connecting ΓåÆ Connected/Failed states)
- Resource handoff: release a completed `CommunicationsChannel` to the caller for bidirectional communication

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect::Status` | enum | Connection state: Connecting, Connected, ResolutionFailed, ConnectFailed |
| `NonblockingConnect` | class | Single async connection attempt with thread and channel management |
| `ConnectPool` | class | Singleton pool of reusable NonblockingConnect objects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ConnectPool::m_instance` | `ConnectPool*` | static | Singleton instance |

## Key Functions / Methods

### NonblockingConnect constructors
- **Signature:** `NonblockingConnect(const std::string& address, uint16 port)` / `NonblockingConnect(const IPaddress& ip)`
- **Purpose:** Initiate asynchronous TCP connection to a hostname/port or IP address
- **Inputs:** Address string + port (host byte order), or pre-resolved IPaddress
- **Outputs/Return:** Object state; actual connection attempt happens on worker thread
- **Side effects:** Spawns SDL thread; creates CommunicationsChannel internally
- **Calls:** `connect()` (private), `connect_thread()` (static)

### NonblockingConnect::status() / done()
- **Signature:** `Status status()` / `bool done()`
- **Purpose:** Poll connection state; `done()` returns true if no longer Connecting
- **Outputs/Return:** Current Status enum / boolean

### NonblockingConnect::address()
- **Signature:** `const IPaddress& address()`
- **Purpose:** Retrieve resolved IP address after successful connection
- **Outputs/Return:** Reference to `m_ip`
- **Notes:** Asserts status is Connected or ResolutionFailed; unsafe to call while Connecting

### NonblockingConnect::release()
- **Signature:** `CommunicationsChannel* release()`
- **Purpose:** Transfer ownership of completed connection channel to caller
- **Outputs/Return:** Pointer to CommunicationsChannel (caller now owns)
- **Side effects:** Relinquishes ownership via `auto_ptr::release()`
- **Notes:** Asserts Connected state; intended final operation before reusing pool object

### ConnectPool::instance()
- **Signature:** `static ConnectPool *instance()`
- **Purpose:** Lazy singleton accessor
- **Outputs/Return:** Global ConnectPool pointer

### ConnectPool::connect() (overloaded)
- **Signature:** `NonblockingConnect* connect(const std::string& address, uint16 port)` / `NonblockingConnect* connect(const IPaddress& ip)`
- **Purpose:** Acquire an unused connection object from pool and initiate async connection
- **Inputs:** Address/port or IPaddress
- **Outputs/Return:** Pointer to reused NonblockingConnect object (now in-use)
- **Side effects:** Marks pool slot as "in use" (pair.second = false)

### ConnectPool::abandon()
- **Signature:** `void abandon(NonblockingConnect*)`
- **Purpose:** Return connection object to pool after connection complete or error
- **Inputs:** Pointer to NonblockingConnect from pool
- **Side effects:** Marks pool slot as available (pair.second = true)

## Control Flow Notes
Connection flow is deferred (non-blocking):
1. Caller invokes `ConnectPool::connect()` ΓåÆ gets a NonblockingConnect object
2. Caller polls `NonblockingConnect::status()` / `done()` in main loop
3. When done (Connected or failed), caller either:
   - Calls `release()` to extract the CommunicationsChannel for I/O
   - Or calls `ConnectPool::abandon()` to return the object to the pool for reuse
4. The actual async work happens on `connect_thread()` (SDL thread context)

## External Dependencies
- **Includes:** `cseries.h` (core types: `uint16`, `IPaddress`), `CommunicationsChannel.h`, `<string>`, `<memory>` (auto_ptr), `<SDL_thread.h>`
- **Defined elsewhere:** `CommunicationsChannel` class (TCPMess module), `IPaddress` type (SDL_net), SDL threading primitives
