# Source_Files/Misc/ActionQueues.cpp

## File Purpose
Implements ActionQueues, a container class for managing circular FIFO queues of player action flags. Encapsulates queue memory allocation, enqueue/dequeue operations, and zombie player control filtering for the Marathon: Aleph One engine.

## Core Responsibilities
- Allocate and manage heap memory for action flag queues across all players
- Enqueue action flags (with optional zombie player filtering)
- Dequeue and peek at action flags in FIFO order
- Count available flags in each queue
- Reset individual or all queues
- Configure zombie player controllability at the queue-set level
- Modify action flags at queue head (ModifiableActionQueues subclass)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `action_queue` | struct | Circular queue metadata: read/write indices and buffer pointer |
| `ActionQueues` | class | Main queue container for managing N players' action queues |
| `ModifiableActionQueues` | class | Extends ActionQueues to allow in-place flag modification |

## Global / File-Static State
None.

## Key Functions / Methods

### ActionQueues (constructor)
- **Signature:** `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- **Purpose:** Allocate memory and initialize queue headers and buffers
- **Inputs:** Number of players, queue diameter (size), zombie controllability flag
- **Outputs/Return:** Object ready for use
- **Side effects:** Heap allocation of `mQueueHeaders` (array of `action_queue`) and `mFlagsBuffer` (flat uint32 array); asserts both succeed; initializes all read/write indices to 0
- **Calls:** `new[]` operator
- **Notes:** Circular buffer model; each player's flags stored contiguously in mFlagsBuffer with stride mQueueSize

### ~ActionQueues (destructor)
- **Signature:** `~ActionQueues()`
- **Purpose:** Free allocated memory
- **Side effects:** Deletes `mFlagsBuffer` and `mQueueHeaders` if non-null
- **Notes:** Safe null-check before deletion

### reset()
- **Signature:** `void reset()`
- **Purpose:** Clear all player queues
- **Side effects:** Resets read/write indices to 0 for all players
- **Notes:** Lifted from player.cpp::reset_player_queues()

### resetQueue(int)
- **Signature:** `void resetQueue(int inPlayerIndex)`
- **Purpose:** Reset a single player's queue
- **Inputs:** Player index
- **Side effects:** Resets read/write indices for one player; asserts valid index
- **Notes:** Added May 14, 2003 for LegacyActionQueueToTickBasedQueueAdapter

### enqueueActionFlags(int, const uint32*, int)
- **Signature:** `void enqueueActionFlags(int player_index, const uint32 *action_flags, int count)`
- **Purpose:** Add action flags to a player's queue
- **Inputs:** Player index, pointer to flags, count of flags to enqueue
- **Side effects:** Advances `write_index` modulo `mQueueSize`; logs error if queue wraps (overflow)
- **Calls:** `get_player_data()`, `logError1()`
- **Notes:** Skips enqueue if `!mZombiesControllable && PLAYER_IS_ZOMBIE(player)`; lifted from player.cpp::queue_action_flags()

### dequeueActionFlags(int)
- **Signature:** `uint32 dequeueActionFlags(int player_index)`
- **Purpose:** Retrieve and remove next action flag from queue
- **Inputs:** Player index
- **Outputs/Return:** Action flag (uint32), or 0 if queue empty or zombie non-controllable
- **Side effects:** Advances `read_index` modulo `mQueueSize`; logs error if dequeuing empty queue
- **Calls:** `get_player_data()`, `logError1()`
- **Notes:** Non-controllable zombies always return 0

### peekActionFlags(int, size_t)
- **Signature:** `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- **Purpose:** Read action flag without removing it
- **Inputs:** Player index, lookahead offset
- **Outputs/Return:** Action flag or 0 if out of bounds
- **Side effects:** None (read-only)
- **Calls:** `get_player_data()`, `countActionFlags()`, `logError3()`
- **Notes:** Added June 14, 2003; index wraps with modulo; logs error if offset >= count

### countActionFlags(int)
- **Signature:** `unsigned int countActionFlags(int player_index)`
- **Purpose:** Return number of queued flags
- **Inputs:** Player index
- **Outputs/Return:** Count (unsigned int)
- **Calls:** `get_player_data()`
- **Notes:** Returns `mQueueSize` for non-controllable zombies (unlimited flags); otherwise `(mQueueSize + write_index - read_index) % mQueueSize`

### setZombiesControllable(bool) / zombiesControllable()
- **Signature:** `void setZombiesControllable(bool)` / `bool zombiesControllable()`
- **Purpose:** Mutator/accessor for zombie controllability setting
- **Side effects:** Updates `mZombiesControllable` member
- **Notes:** Made a queue-set property rather than per-method argument (Feb 3, 2003)

### ModifiableActionQueues::modifyActionFlags(int, uint32, uint32)
- **Signature:** `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- **Purpose:** In-place modification of flags at queue head
- **Inputs:** Player index, flags to apply, bitmask (which bits to modify)
- **Side effects:** Modifies `buffer[read_index]` in-place; returns early if queue empty
- **Calls:** `countActionFlags()`, `logError()`
- **Notes:** Bitwise operation: `(flags & ~mask) | (inFlags & mask)`; skips if sentinel 0xffffffff detected

## Control Flow Notes
Not directly inferable as a lifecycle component. These queues are populated by input/network layers and drained by `update_players()` during game ticks to feed player movement/action logic.

## External Dependencies
- **player.h:** `get_player_data()`, `PLAYER_IS_ZOMBIE()` macro, player_data struct
- **Logging.h:** `logError1()`, `logError3()`, `logError()` macros
- **cseries.h:** Basic types (uint32, assert, etc.)
