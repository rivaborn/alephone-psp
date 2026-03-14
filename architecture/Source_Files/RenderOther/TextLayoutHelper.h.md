# Source_Files/RenderOther/TextLayoutHelper.h

## File Purpose
Utility class for managing non-overlapping rectangular space allocations. Provides a simple reservation system to place text or UI elements without overlap, tracking used regions and computing safe placement coordinates.

## Core Responsibilities
- Reserve rectangular space with guaranteed non-overlap detection
- Track horizontal and vertical boundaries of all reservations
- Compute minimum Y-coordinate that avoids existing reservations
- Clear and reset all active reservations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TextLayoutHelper` | class | Main layout manager |
| `Reservation` | struct | Stores vertical bounds (top/bottom) of a reserved region |
| `ReservationEnd` | struct | Tracks a single vertical boundary with its parent reservation |
| `CollectionOfReservationEnds` | typedef | Vector of ReservationEnd entries |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mReservationEnds` | `CollectionOfReservationEnds` | member | Stores all active reservation boundaries for collision detection |

## Key Functions / Methods

### reserveSpaceFor
- **Signature:** `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- **Purpose:** Reserve space for a new rectangle and compute safe Y-placement without overlap
- **Inputs:** 
  - `inLeft`: left X-coordinate
  - `inWidth`: rectangle width
  - `inLowestBottom`: minimum acceptable Y-coordinate for bottom edge
  - `inHeight`: rectangle height
- **Outputs/Return:** Smallest bottom Y-coordinate that avoids all prior reservations
- **Side effects:** Adds new reservation boundaries to `mReservationEnds`
- **Notes:** Client responsible for actually placing rectangle at returned coordinate; algorithm checks horizontal overlap to determine if vertical constraints apply

### removeAllReservations
- **Signature:** `void removeAllReservations()`
- **Purpose:** Clear all active reservations
- **Side effects:** Empties `mReservationEnds`

## Control Flow Notes
Used during rendering/UI layout phases to position text elements or overlays without collision. Called iteratively when laying out multiple elements sequentially (e.g., on-screen messages, debug text).

## External Dependencies
- `<vector>` (STL container for tracking reservation boundaries)
- `using namespace std;`
