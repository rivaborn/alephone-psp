# Source_Files/Misc/CircularByteBuffer.cpp

## File Purpose
Implementation of a circular queue for bytes with support for bulk operations. Provides both copy-based and zero-copy (direct pointer) interfaces for enqueueing and peeking data, correctly handling wraparound at the buffer boundary.

## Core Responsibilities
- Split circular buffer operations into up to two contiguous chunks to handle wraparound
- Copy bytes into the buffer (enqueue) and out of the buffer (peek)
- Expose direct buffer pointers for zero-copy enqueueing operations (start/finish pattern)
- Expose direct buffer pointers for zero-copy peeking operations
- Assert preconditions (sufficient space for enqueue, sufficient data for peek)
- Advance read/write indices after operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `std::pair<unsigned int, unsigned int>` | typedef | Return type for chunk split results (first chunk size, second chunk size) |

## Global / File-Static State
None.

## Key Functions / Methods

### splitIntoChunks
- **Signature:** `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- **Purpose:** Calculate how to split a read/write operation across the circular buffer's boundary when it wraps.
- **Inputs:** Byte count needed, starting index (read or write), total buffer size.
- **Outputs/Return:** Pair of (first chunk size, second chunk size); sum equals `inByteCount`.
- **Side effects:** None.
- **Calls:** `std::min()`.
- **Notes:** If no wraparound needed, second chunk is 0. Static utility usable by external clients.

### enqueueBytes
- **Signature:** `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- **Purpose:** Copy bytes into the buffer, handling wraparound via `memcpy` into two chunks if needed.
- **Inputs:** Source buffer pointer, byte count to copy.
- **Outputs/Return:** None (modifies buffer in-place).
- **Side effects:** Writes to `mData`; advances `mWriteIndex` via `advanceWriteIndex()`.
- **Calls:** `splitIntoChunks()`, `memcpy()`, `advanceWriteIndex()`.
- **Notes:** Asserts `inByteCount <= getRemainingSpace()`. Early return if count is 0.

### enqueueBytesNoCopyStart
- **Signature:** `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Return direct pointers into the buffer where caller should write data, avoiding a copy.
- **Inputs:** Requested byte count; output parameter pointers (any may be NULL).
- **Outputs/Return:** Fills output pointers with addresses and counts for up to two writable chunks.
- **Side effects:** None; does not advance write index.
- **Calls:** `splitIntoChunks()`, `getWriteIndex()`.
- **Notes:** Asserts `inByteCount <= getRemainingSpace()`. Caller responsible for writing and calling `enqueueBytesNoCopyFinish()` afterward.

### enqueueBytesNoCopyFinish
- **Signature:** `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- **Purpose:** Commit the write operation started by `enqueueBytesNoCopyStart()` by advancing the write index.
- **Inputs:** Number of bytes actually written.
- **Outputs/Return:** None.
- **Side effects:** Advances `mWriteIndex` via `advanceWriteIndex()`.
- **Calls:** `advanceWriteIndex()`.
- **Notes:** Caller may write fewer bytes than initially requested; specify actual count here.

### peekBytes
- **Signature:** `void peekBytes(void* outBytes, unsigned int inByteCount)`
- **Purpose:** Copy bytes from the buffer without removing them, handling wraparound.
- **Inputs:** Destination buffer pointer, byte count to copy.
- **Outputs/Return:** Copies data to `outBytes`.
- **Side effects:** None; does not advance read index.
- **Calls:** `splitIntoChunks()`, `memcpy()`.
- **Notes:** Asserts `inByteCount <= getCountOfElements()`. Early return if count is 0.

### peekBytesNoCopy
- **Signature:** `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Return direct pointers into the buffer for reading, avoiding a copy.
- **Inputs:** Requested byte count; output parameter pointers (any may be NULL).
- **Outputs/Return:** Fills output pointers with const addresses and counts for up to two readable chunks.
- **Side effects:** None; does not advance read index.
- **Calls:** `splitIntoChunks()`, `getReadIndex()`.
- **Notes:** Asserts `inByteCount <= getCountOfElements()`. Caller may read fewer bytes than requested and call `dequeue()` afterward.

## Control Flow Notes
Not part of frame/update/render loop. Utility data structure called by other systems (likely network I/O or other byte-stream consumers). Two-phase zero-copy API allows callers to pair with `writev()`/`readv()`-style system calls.

## External Dependencies
- **Includes:** `cseries.h` (for `assert`), `CircularByteBuffer.h`, `<algorithm>` (for `std::min`), `<utility>` (via header).
- **Defined elsewhere:** `CircularQueue<char>` (base class), `mData`, `mWriteIndex`, `mReadIndex`, `mQueueSize`, `advanceWriteIndex()`, `getWriteIndex()`, `getReadIndex()`, `getRemainingSpace()`, `getCountOfElements()`.
