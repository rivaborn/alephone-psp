# Source_Files/Misc/CircularQueue.h

## File Purpose
Template class implementing a generic circular queue (ring buffer) data structure for the Aleph One engine. Provides fixed-capacity, wrap-around storage with O(1) enqueue/dequeue operations and supports copy semantics.

## Core Responsibilities
- Allocate and manage a fixed-size circular buffer for arbitrary element types
- Provide enqueue (push) and dequeue (pop) operations with modulo-arithmetic indexing
- Track read and write pointers to support wrap-around behavior
- Support resizing and deep copy of queue state
- Assert invariants (no over-enqueue, no over-dequeue)

## Key Types / Data Structures
None (standalone template class).

## Global / File-Static State
None.

## Key Functions / Methods

### CircularQueue (constructors)
- Signature: `CircularQueue()`, `explicit CircularQueue(unsigned int inSize)`, `CircularQueue(const CircularQueue<T>& o)`
- Purpose: Initialize queue with optional size, or copy another queue
- Inputs: `inSize` (capacity in elements), `o` (queue to copy)
- Outputs/Return: None
- Side effects: Allocates `inSize + 1` elements on heap
- Calls: `reset(inSize)`, `operator=`
- Notes: Copy constructor delegates to assignment operator; size+1 storage allows distinguishing empty from full

### reset
- Signature: `void reset(unsigned int inSize)` / `void reset()`
- Purpose: Resize and clear the queue to a new capacity
- Inputs: `inSize` (new capacity; default = current total space)
- Outputs/Return: None
- Side effects: Deallocates old buffer, allocates new `inSize + 1` elements, resets pointers to zero
- Calls: `getTotalSpace()`, `new[]`, `delete[]`
- Notes: Asserts wrap-around will not occur (`theStorageCount > inSize`)

### enqueue
- Signature: `void enqueue(const T& inData)`
- Purpose: Add element to back of queue
- Inputs: `inData` (element to queue)
- Outputs/Return: None
- Side effects: Advances write pointer
- Calls: `getWriteIndex()`, `advanceWriteIndex()`
- Notes: Asserts remaining space > 0 (via `advanceWriteIndex`)

### dequeue
- Signature: `void dequeue(unsigned int inAmount = 1)`
- Purpose: Remove one or more elements from front of queue
- Inputs: `inAmount` (count to dequeue; default 1)
- Outputs/Return: None
- Side effects: Advances read pointer
- Calls: `advanceReadIndex()`
- Notes: Asserts sufficient elements exist

### peek
- Signature: `const T& peek() const`
- Purpose: Read front element without removing
- Inputs: None
- Outputs/Return: Const reference to element at read index
- Side effects: None
- Calls: `getReadIndex()`

### Utility accessors
- `getCountOfElements()` ΓÇô returns elements currently in queue via modulo arithmetic
- `getRemainingSpace()` ΓÇô returns available slots (getTotalSpace() - count)
- `getTotalSpace()` ΓÇô returns max capacity (mQueueSize - 1)

**Protected helpers** (internal):
- `getReadIndex(offset)`, `getWriteIndex(offset)` ΓÇô compute modulo indices with assertions
- `advanceReadIndex(amount)`, `advanceWriteIndex(amount)` ΓÇô move pointers with wrap-around and capacity checks

### operator= (assignment)
- Signature: `CircularQueue<T>& operator=(const CircularQueue<T>& o)`
- Purpose: Deep copy another queue's state and contents
- Inputs: `o` (source queue)
- Outputs/Return: Reference to this
- Side effects: Resizes this queue, enqueues copies of all elements from `o`
- Calls: `reset()`, `getCountOfElements()`, `getReadIndex()`, `enqueue()`
- Notes: Self-assignment guarded; iterates by index to handle potential pointer differences

## Control Flow Notes
Utility data structure; not tied to engine frame/render loop. Likely used for buffering network packets, input events, or other deferred processing in the multiplayer Marathon engine.

## External Dependencies
- Standard C++ (`assert`, `new[]`, `delete[]`)
- No engine-specific headers included
