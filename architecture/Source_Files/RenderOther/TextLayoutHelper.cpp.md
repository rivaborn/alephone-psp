# Source_Files/RenderOther/TextLayoutHelper.cpp

## File Purpose
Implements a layout helper that manages placement of non-overlapping rectangles (likely for UI/HUD text rendering in Marathon: Aleph One). Given a new rectangle's horizontal extent and minimum vertical position, it calculates a safe vertical position that avoids collisions with previously reserved rectangles.

## Core Responsibilities
- Track active rectangle reservations via horizontal/vertical boundary events
- Insert new reservations into a sorted list of horizontal boundaries
- Detect overlapping rectangles by horizontal extent
- Resolve vertical collisions by adjusting placement upward iteratively
- Manage memory cleanup for all tracked reservations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Reservation` | struct | Represents a single reserved rectangle; stores vertical bounds (`mTop`, `mBottom`) |
| `ReservationEnd` | struct | Marks the start or end of a reservation's horizontal extent; stores `mHorizontalCoordinate`, pointer to `Reservation`, and a flag indicating start vs. end |
| `CollectionOfReservationEnds` | typedef | `vector<ReservationEnd>` ΓÇö maintains sorted list of horizontal boundary events |

## Global / File-Static State
None.

## Key Functions / Methods

### reserveSpaceFor
- **Signature:** `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- **Purpose:** Allocate space for a rectangle at the given horizontal position and height, calculating the smallest non-overlapping vertical position.
- **Inputs:**
  - `inLeft`: left edge of desired rectangle
  - `inWidth`: width in pixels
  - `inLowestBottom`: bottom edge constraint (minimum allowed position)
  - `inHeight`: height in pixels
- **Outputs/Return:** The calculated bottom edge coordinate for the rectangle (ΓëÑ `inLowestBottom`)
- **Side effects:** 
  - Allocates new `Reservation` object (heap)
  - Inserts two `ReservationEnd` entries into `mReservationEnds` vector (maintains sorted order by `mHorizontalCoordinate`)
  - Modifies `Reservation` fields (`mTop`, `mBottom`)
- **Calls (direct):** `std::multiset::insert()`, `std::vector::insert()`, `std::vector::end()`, `std::vector::begin()`
- **Notes:** 
  - Algorithm: walks the sorted boundary list twice ΓÇö once to track overlapping reservations at the left edge, once to include all reservations that span across the rectangle's horizontal extent. Then iteratively checks vertical overlap and nudges the rectangle upward until clear.
  - Includes casts and assertions to validate `inHeight` is safe.
  - Complexity is O(n┬▓) in worst case due to the do-while loop re-scanning overlaps on each vertical adjustment.

### removeAllReservations
- **Signature:** `void removeAllReservations()`
- **Purpose:** Clean up all tracked reservations and free heap memory.
- **Side effects:** Deletes all `Reservation` objects; clears `mReservationEnds` vector.
- **Calls:** `std::vector::begin()`, `std::vector::end()`, `std::vector::clear()`
- **Notes:** Iterates through start/end pairs and deletes each `Reservation` exactly once (using `mStartOfReservation` flag to skip end markers).

### Trivial methods
- **TextLayoutHelper()**: Empty constructor.
- **~TextLayoutHelper()**: Calls `removeAllReservations()` in destructor.

## Control Flow Notes
- **Initialization:** Constructor is empty; instance created with empty `mReservationEnds`.
- **Frame/Update:** `reserveSpaceFor()` called repeatedly (likely once per HUD element per frame) to lay out non-overlapping text/UI boxes.
- **Shutdown:** Destructor ensures all allocations are freed.
- Likely integrated into the render pipeline for UI/HUD layout before drawing.

## External Dependencies
- `<vector>` ΓÇö `CollectionOfReservationEnds` storage
- `<set>` ΓÇö `std::multiset<Reservation*>` for tracking overlaps
- `<assert.h>` ΓÇö runtime bounds/sanity checks
- `TextLayoutHelper.h` ΓÇö type definitions (`Reservation`, `ReservationEnd`)
