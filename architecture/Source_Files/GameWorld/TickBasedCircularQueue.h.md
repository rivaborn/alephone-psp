# Source_Files/GameWorld/TickBasedCircularQueue.h

## File Purpose
Header-only template library implementing lock-free circular queues for tick-based game state buffering. Provides a single-reader, single-writer FIFO keyed by game tick, commonly used to queue action flags across game updates without explicit synchronization.

## Core Responsibilities
- Define a thread-safe circular queue abstraction for tick-keyed game elements (e.g., action flags)
- Provide abstract interface `WritableTickBasedCircularQueue` for queue writers
- Implement concrete queue `ConcreteTickBasedCircularQueue` with dual-tick (read/write) management
- Implement `DuplicatingTickBasedCircularQueue` to fan-out writes to multiple child queues
- Support safe concurrent reader/writer access without locks (via atomic int32 operations)
- Manage circular buffer with proper wrapping for negative and large tick values
- Allow post-enqueue element mutation via `MutableElementsTickBasedCircularQueue`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WritableTickBasedCircularQueue<tValueType>` | Template class (abstract interface) | Base interface for any queue writer; defines reset, enqueue, getWriteTick, availableCapacity |
| `DuplicatingTickBasedCircularQueue<tValueType>` | Template class (mixin/adapter) | Fan-out writer that broadcasts enqueue calls to multiple child queues; returns min capacity across children |
| `ConcreteTickBasedCircularQueue<tValueType>` | Template class (concrete) | Full circular queue implementation with read/write ticks, buffer management, and copy constructor |
| `MutableElementsTickBasedCircularQueue<tValueType>` | Template class (subclass) | Variant that exposes `at(tick)` and `operator[]` to allow mutation of elements after enqueueing |

## Global / File-Static State
None.

## Key Functions / Methods

### WritableTickBasedCircularQueue::reset
- **Signature:** `virtual void reset(int32 inTick) = 0`
- **Purpose:** Reset queue state to a given tick; primarily reader and writer synchronization point
- **Inputs:** `inTick` ΓÇô initial tick value
- **Outputs/Return:** None
- **Notes:** Pure virtual; implemented by subclasses

### WritableTickBasedCircularQueue::enqueue
- **Signature:** `virtual void enqueue(const tValueType& inFlags) = 0`
- **Purpose:** Enqueue one element at the current write tick; for writer only
- **Inputs:** `inFlags` ΓÇô element to append
- **Outputs/Return:** None
- **Notes:** Pure virtual; writer must check `availableCapacity() > 0` first

### WritableTickBasedCircularQueue::availableCapacity
- **Signature:** `virtual int32 availableCapacity() const = 0`
- **Purpose:** Return number of free slots in the queue
- **Notes:** Writer checks this before enqueueing; thread-safe because reader may only *decrease* it

### WritableTickBasedCircularQueue::getWriteTick
- **Signature:** `virtual int32 getWriteTick() const = 0`
- **Purpose:** Return the next tick to be written
- **Notes:** Advances with each enqueue

---

### DuplicatingTickBasedCircularQueue::reset
- **Signature:** `void reset(int32 inTick)`
- **Purpose:** Reset all child queues to the same tick
- **Calls:** Calls `reset(inTick)` on each child in `mChildren`
- **Notes:** Iterates over `std::set` of children; not concurrent-safe if children collection is modified

### DuplicatingTickBasedCircularQueue::availableCapacity
- **Signature:** `int32 availableCapacity() const`
- **Purpose:** Return bottleneck capacity: the minimum available capacity among all children
- **Outputs/Return:** Minimum of all child queue capacities; `INT_MAX` if empty
- **Notes:** Conservative: ensures we never overflow the slowest (most-read-behind) child

### DuplicatingTickBasedCircularQueue::enqueue
- **Signature:** `void enqueue(const tValueType& inFlags)`
- **Purpose:** Broadcast enqueue to all child queues
- **Calls:** `enqueue(inFlags)` on each child
- **Notes:** Atomicity of broadcast depends on writer not being interrupted mid-loop

---

### ConcreteTickBasedCircularQueue::ConcreteTickBasedCircularQueue (constructor)
- **Signature:** `ConcreteTickBasedCircularQueue(int inBufferCapacity)`
- **Purpose:** Allocate and initialize circular buffer
- **Inputs:** `inBufferCapacity` ΓÇô desired queue depth
- **Side effects:** Allocates `new tValueType[inBufferCapacity + 1]` on heap; calls `reset(0)`
- **Notes:** Buffer oversized by 1 to distinguish full from empty

### ConcreteTickBasedCircularQueue::ConcreteTickBasedCircularQueue (copy constructor)
- **Signature:** `ConcreteTickBasedCircularQueue(const ConcreteTickBasedCircularQueue<tValueType>& o)`
- **Purpose:** Deep copy queue state and elements
- **Inputs:** `o` ΓÇô source queue
- **Side effects:** Allocates new buffer; copies all enqueued elements using `peek()`
- **Notes:** Safe only when neither reader nor writer is active; iterates `[readTick, writeTick)`

### ConcreteTickBasedCircularQueue::reset
- **Signature:** `void reset(int32 inTick)`
- **Purpose:** Clear queue and set read/write ticks to the same value
- **Side effects:** `mReadTick = mWriteTick = inTick` (does not clear buffer memory)

### ConcreteTickBasedCircularQueue::size
- **Signature:** `int32 size() const`
- **Purpose:** Return number of enqueued elements
- **Outputs/Return:** `mWriteTick - mReadTick`

### ConcreteTickBasedCircularQueue::availableCapacity
- **Signature:** `int32 availableCapacity() const`
- **Purpose:** Return free slots
- **Outputs/Return:** `totalCapacity() - size()`

### ConcreteTickBasedCircularQueue::peek
- **Signature:** `const tValueType& peek(int32 inTick) const`
- **Purpose:** Read element at arbitrary tick without advancing read position
- **Inputs:** `inTick` ΓÇô tick to read
- **Outputs/Return:** Reference to element
- **Calls:** `elementForTick(inTick)`
- **Notes:** Reader-only; asserts `inTick` in `[readTick, writeTick)`

### ConcreteTickBasedCircularQueue::dequeue
- **Signature:** `void dequeue()`
- **Purpose:** Advance read tick by 1
- **Side effects:** Increments `mReadTick`
- **Notes:** Reader-only; asserts `size() > 0`

### ConcreteTickBasedCircularQueue::enqueue
- **Signature:** `void enqueue(const tValueType& inFlags)`
- **Purpose:** Append element at write tick and advance write pointer
- **Inputs:** `inFlags` ΓÇô element to add
- **Side effects:** Updates buffer at `(mWriteTick % mBufferSize)`, increments `mWriteTick`
- **Calls:** `elementForTick()` indirectly (writes to buffer)
- **Notes:** Writer-only; asserts `availableCapacity() > 0`; normalizes `mWriteTick` to positive before mod to handle wraparound

### ConcreteTickBasedCircularQueue::elementForTick (protected)
- **Signature:** `tValueType& elementForTick(int32 inTick) const`
- **Purpose:** Internal helper to get buffer slot for a given tick
- **Inputs:** `inTick` ΓÇô desired tick
- **Outputs/Return:** Reference to buffer element
- **Notes:** Normalizes negative ticks by repeatedly adding `mBufferSize` until positive, then uses modulo arithmetic; asserts `inTick` in valid range

---

### MutableElementsTickBasedCircularQueue::at
- **Signature:** `tValueType& at(int32 inTick)`
- **Purpose:** Mutable access to element at tick (allows post-enqueue mutation)
- **Inputs:** `inTick` ΓÇô element tick
- **Outputs/Return:** Mutable reference
- **Calls:** `ConcreteTickBasedCircularQueue::elementForTick(inTick)`
- **Notes:** Requires careful synchronization; reader must not read while writer modifies

### MutableElementsTickBasedCircularQueue::operator[]
- **Signature:** `tValueType& operator[](int32 inTick)`
- **Purpose:** Array-like mutable access; convenience alias for `at()`
- **Calls:** `at(inTick)`

## Control Flow Notes
This is a utility data structure (not part of the main game loop) used to decouple game tick producers from consumers. Typically:
1. **Writer** calls `enqueue()` each tick (after checking capacity).
2. **Reader** calls `peek()` and `dequeue()` independently, possibly at a different rate.
3. No locks required because int32 read/write is atomic on most platforms; the invariants in file comments ensure safety:
   - Writer cannot overflow because it checks capacity.
   - Reader cannot underflow because it checks size.
   - Behind writer's back, `availableCapacity()` only increases.
   - Behind reader's back, `size()` only increases.

## External Dependencies
- **cseries.h** ΓÇô Provides `int32`, assert macros, and basic type definitions
- **std::set** ΓÇô Used in `DuplicatingTickBasedCircularQueue::mChildren` to track child queue pointers
