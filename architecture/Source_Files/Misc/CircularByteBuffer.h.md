# Source_Files/Misc/CircularByteBuffer.h

## File Purpose
A circular FIFO byte buffer extending `CircularQueue<char>` with mass-data operations and zero-copy access patterns. Designed to efficiently transfer bytes with wraparound handling, including support for direct buffer pointers to avoid unnecessary copies.

## Core Responsibilities
- Provide FIFO byte queueing with proper wraparound semantics
- Support bulk peek/enqueue operations for byte chunks
- Expose zero-copy "NoCopy" interfaces for performance-critical code paths
- Handle buffer "seam" splitting when data wraps around the circular boundary
- Maintain safe invariants (e.g., caller must verify space/data availability)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularByteBuffer` | class | Main circular byte buffer with convenience methods |
| `CircularByteBufferBase` | typedef | `CircularQueue<char>` base type |

## Global / File-Static State
None.

## Key Functions / Methods

### CircularByteBuffer (constructor)
- Signature: `CircularByteBuffer(unsigned int inUsableBytes)`
- Purpose: Initialize buffer with capacity for the specified number of usable bytes
- Inputs: `inUsableBytes` ΓÇô desired queue capacity
- Outputs/Return: None
- Side effects: Allocates internal storage via parent `CircularQueue`
- Calls: `CircularByteBufferBase` constructor
- Notes: Actual storage is `inUsableBytes + 1` to distinguish empty from full

### peekBytes
- Signature: `void peekBytes(void* outBytes, unsigned int inByteCount)`
- Purpose: Copy queued bytes to caller's buffer without removing them
- Inputs: `outBytes` pointer, `inByteCount` bytes to read
- Outputs/Return: Data copied to `outBytes`
- Side effects: None (read-only)
- Calls: Not inferable from header
- Notes: Caller must verify `inByteCount <= getCountOfElements()`; handles wraparound internally

### enqueueBytes
- Signature: `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- Purpose: Copy bytes from caller's buffer into the queue
- Inputs: `inBytes` pointer, `inByteCount` bytes to write
- Outputs/Return: None
- Side effects: Advances write index; modifies queue contents
- Calls: Not inferable from header
- Notes: Caller must verify `inByteCount <= getRemainingSpace()`; handles wraparound

### peekBytesNoCopy
- Signature: `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Return direct pointers to readable buffer regions without copying
- Inputs: `inByteCount` bytes requested
- Outputs/Return: Up to two buffer pointers and their lengths (for wrapping case)
- Side effects: None; read index not advanced
- Calls: Not inferable from header
- Notes: Second chunk is NULL/zero if wraparound unnecessary; read index advanced only by subsequent `dequeue()`; pointers are const

### enqueueBytesNoCopyStart
- Signature: `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- Purpose: Begin zero-copy enqueue by exposing writable buffer regions
- Inputs: `inByteCount` bytes to enqueue
- Outputs/Return: Up to two writable buffer pointers and available lengths
- Side effects: **Prepares** for write but does **not** advance write index
- Calls: Not inferable from header
- Notes: Must call `enqueueBytesNoCopyFinish()` after writing; caller may write fewer bytes than requested

### enqueueBytesNoCopyFinish
- Signature: `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- Purpose: Commit a zero-copy enqueue operation by advancing the write index
- Inputs: `inActualByteCount` ΓÇô bytes actually written (may be Γëñ start request)
- Outputs/Return: None
- Side effects: Advances write index
- Calls: Not inferable from header
- Notes: Must be paired with `enqueueBytesNoCopyStart()`; decoupling write and index update ensures data stability

### splitIntoChunks (static)
- Signature: `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- Purpose: Compute chunk boundaries when data wraps around the circular buffer seam
- Inputs: `inByteCount`, `inStartingIndex`, `inQueueSize`
- Outputs/Return: `std::pair<uint, uint>` ΓÇô lengths of first and second chunks (second is 0 if no wrap)
- Side effects: None
- Calls: Not inferable from header
- Notes: Static utility; reusable by external clients; first + second = `inByteCount`

## Control Flow Notes
This is a header declaring a data structure; control flow is determined by callers. The class provides both safe (copy-based) and unsafe (zero-copy) access patterns to support both general use and performance-critical paths (e.g., writev/readv integration).

## External Dependencies
- `<utility>` ΓÇô `std::pair`
- `"CircularQueue.h"` ΓÇô template base class `CircularQueue<T>` providing core queue mechanics (mData, mReadIndex, mWriteIndex)
