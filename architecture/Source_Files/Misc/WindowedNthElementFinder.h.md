# Source_Files/Misc/WindowedNthElementFinder.h

## File Purpose
A C++ template utility for efficiently finding the nth smallest or nth largest element within a fixed-size sliding window of recently inserted elements. Commonly used for statistics (e.g., median, percentile calculations) in gameplay or diagnostics systems.

## Core Responsibilities
- Maintain a fixed-size sliding window of elements using a circular queue
- Keep elements sorted for O(log n) insertion and O(1) nth-element queries
- Automatically evict the oldest element when the window reaches capacity
- Provide 0-indexed access to the nth smallest and nth largest elements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mQueue` | `CircularQueue<tElementType>` | FIFO buffer tracking insertion order and managing window size |
| `mSortedElements` | `std::multiset<tElementType>` | Sorted multiset for O(log n) ordered access and duplicate handling |

## Global / File-Static State
None.

## Key Functions / Methods

### `WindowedNthElementFinder()`
- **Signature:** `WindowedNthElementFinder()`, `explicit WindowedNthElementFinder(unsigned int inWindowSize)`
- **Purpose:** Construct with default (size 0) or specified window size
- **Inputs:** Optional window size in elements
- **Outputs/Return:** N/A (constructor)
- **Side effects:** Allocates circular queue storage

### `insert(const tElementType&)`
- **Signature:** `void insert(const tElementType& inNewElement)`
- **Purpose:** Add a new element to the window; evict oldest if window is full
- **Inputs:** Reference to element to insert
- **Outputs/Return:** None
- **Side effects:** Modifies both `mQueue` and `mSortedElements`; may deallocate/reallocate queue storage
- **Calls:** `window_full()`, `peek()`, `dequeue()`, `enqueue()`, multiset `erase()` / `insert()`
- **Notes:** Duplicate elements are allowed (multiset). Eviction is FIFO (oldest first).

### `nth_smallest_element(unsigned int n)`
- **Signature:** `const tElementType& nth_smallest_element(unsigned int n)`
- **Purpose:** Return the nth smallest element (0-indexed; n=0 is minimum)
- **Inputs:** Zero-based index
- **Outputs/Return:** Const reference to the element
- **Side effects:** None
- **Calls:** Multiset `begin()`, iterator arithmetic
- **Notes:** Asserts `n < size()`. O(n) worst-case traversal due to linear iterator advance.

### `nth_largest_element(unsigned int n)`
- **Signature:** `const tElementType& nth_largest_element(unsigned int n)`
- **Purpose:** Return the nth largest element (0-indexed; n=0 is maximum)
- **Inputs:** Zero-based index
- **Outputs/Return:** Const reference to the element
- **Side effects:** None
- **Calls:** Multiset `rbegin()`, reverse iterator arithmetic
- **Notes:** Asserts `n < size()`. O(n) worst-case traversal due to linear iterator advance.

### Accessors
- **`window_full()` / `size()` / `window_size()`:** Query current state; delegate to underlying queue.

## Control Flow Notes
This is a utility class with no implicit game-loop integration. Typical use: inserted into per-frame updates to accumulate metrics (latency, frametime, etc.), then queried on-demand or during periodic diagnostics (HUD overlay, logging).

## External Dependencies
- **Included:** `CircularQueue.h` (custom circular queue template)
- **Implicit STL dependencies:** `<set>` (for `std::multiset`), `<cassert>` (for assertions)
- **Defined elsewhere:** GPL license header references to COPYING file
