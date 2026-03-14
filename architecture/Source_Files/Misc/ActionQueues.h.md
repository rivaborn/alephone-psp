# Source_Files/Misc/ActionQueues.h

## File Purpose
Defines a circular queue system for managing player action flags across multiple players in a networked multiplayer game context. Encapsulates enqueueing/dequeueing input actions and supports controllable zombie (AI) players. Part of the Marathon: Aleph One networking/input system.

## Core Responsibilities
- Manage circular queues of action flags for each player
- Enqueue and dequeue player action flags with overflow/capacity tracking
- Peek at queued actions without removing them
- Reset individual or all player queues
- Track and configure zombie AI controllability per queue-set
- Support modification of action flags at queue head (subclass)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueues` | class | Main queue manager for multiple players |
| `ModifiableActionQueues` | class | Extends ActionQueues with flag modification capability |
| `action_queue` | struct (nested) | Per-player queue with read/write indices and buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- Purpose: Initialize action queue system for a given player count
- Inputs: Number of players, queue size per player, zombie controllability flag
- Outputs/Return: Initialized object
- Side effects: Allocates `mFlagsBuffer` (shared buffer) and `mQueueHeaders` array
- Calls: (allocation via new/delete, inferred)
- Notes: Copy constructor and assignment are explicitly hidden

### enqueueActionFlags
- Signature: `void enqueueActionFlags(int inPlayerIndex, const uint32* inFlags, int inFlagsCount)`
- Purpose: Add action flags to a player's queue
- Inputs: Player index, array of flag values, count of flags
- Outputs/Return: None
- Side effects: Advances write_index in target queue; may fail silently if queue full
- Calls: (queue write logic, implementation in .cpp)
- Notes: Respects `mZombiesControllable` setting

### dequeueActionFlags
- Signature: `uint32 dequeueActionFlags(int inPlayerIndex)`
- Purpose: Retrieve and remove oldest action flag from queue
- Inputs: Player index
- Outputs/Return: Next action flag value
- Side effects: Advances read_index; undefined if queue empty
- Calls: (queue read logic, implementation in .cpp)
- Notes: Returns single uint32; respects zombie controllability

### peekActionFlags
- Signature: `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- Purpose: Examine queued flags without removing them
- Inputs: Player index, offset from queue head
- Outputs/Return: Action flag value at that position
- Side effects: None (read-only)
- Calls: (queue access logic, implementation in .cpp)
- Notes: Added June 14, 2003; allows inspection without dequeue

### Capacity / Inspection Methods
- `countActionFlags()` ΓÇö returns number of queued flags
- `totalCapacity()` ΓÇö returns `mQueueSize - 1` (circular queue reserve)
- `availableCapacity()` ΓÇö returns free slots
- `zombiesControllable() / setZombiesControllable()` ΓÇö getter/setter for zombie AI control flag

### ModifiableActionQueues::modifyActionFlags
- Signature: `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- Purpose: Modify action flags at the head of the queue in-place
- Inputs: Player index, new flag bits, mask for which bits to change
- Outputs/Return: None
- Side effects: Mutates head element; does not dequeue
- Calls: (queue modification, implementation in .cpp)
- Notes: Subclass-only method; used to alter pending actions

## Control Flow Notes
This class integrates into the game's input/action pipeline:
- During frame input collection, player actions are **enqueued** via `enqueueActionFlags()`
- During the game update/logic phase, actions are **dequeued** via `dequeueActionFlags()` and processed
- `peekActionFlags()` supports queries without consuming the action
- `modifyActionFlags()` (subclass) allows dynamic action tweaking before processing

Fits into the frame/update loop between input capture and action execution.

## External Dependencies
- `cseries.h` ΓÇö standard utility definitions (includes SDL, type definitions, cross-platform macros)
- Standard C++ types: `uint32`, `size_t`, `bool`
- No other game engine symbols visible (queue implementation in paired .cpp file)

---

**Notes:**
- Copy constructor and assignment operator are deliberately hidden (`private`) to prevent shallow copies
- Actual queue buffer management (allocation, write/read logic) deferred to implementation file
- Zombie controllability evolved from per-call parameter (2002) to per-queue-set property (2003)
