# Source_Files/RenderOther/TextLayoutHelper.cpp

## File Purpose
Implements a rectangle layout helper that tracks and reserves non-overlapping rectangular regions in 2D space. It provides functionality to position new rectangles without colliding with existing reservations, primarily for UI text/element placement.

## Core Responsibilities
- Maintain a sorted collection of rectangle boundaries indexed by horizontal coordinate
- Track which reservations overlap in X-space with newly positioned rectangles
- Calculate the lowest valid Y-position for a new rectangle given constraints
- Manage memory for dynamically allocated reservation objects
- Iterate vertically upward until finding a non-overlapping position

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Reservation` | struct | Stores vertical bounds (`mTop`, `mBottom`) of a reserved rectangle |
| `ReservationEnd` | struct | Represents a point where a reservation starts/ends in X-space; tracks horizontal coordinate and whether it's a left (start) or right (end) boundary |
| `CollectionOfReservationEnds` | typedef | Vector of ReservationEnd structs, sorted by horizontal coordinate |
| `CollectionOfReservationPointers` | typedef (local) | Multiset of Reservation pointers tracking active overlaps in X-space |

## Global / File-Static State
None.

## Key Functions / Methods

### reserveSpaceFor
- **Signature:** `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- **Purpose:** Find the lowest valid Y-coordinate for a new rectangle that does not overlap existing reservations.
- **Inputs:**
  - `inLeft`: left X-coordinate of desired rectangle
  - `inWidth`: width (right = inLeft + inWidth)
  - `inLowestBottom`: minimum acceptable Y-bottom
  - `inHeight`: rectangle height
- **Outputs/Return:** The calculated bottom Y-coordinate (always ΓëÑ inLowestBottom if possible)
- **Side effects:** Allocates new `Reservation` object; inserts two `ReservationEnd` entries into `mReservationEnds` (maintains X-sorted order); modifies global reservation state
- **Calls:** `std::vector::insert()`, `std::multiset::insert()`, `std::multiset::erase()`, `new Reservation`
- **Notes:** 
  - Algorithm walks `mReservationEnds` to identify all reservations overlapping the X-range `[inLeft, inLeft+inWidth)`
  - Uses nested loop: repeatedly checks overlapping reservations and adjusts `theCurrentBottom` upward until no conflicts exist
  - Inefficient but functional; acknowledged in header comments as not optimal

### removeAllReservations
- **Signature:** `void removeAllReservations()`
- **Purpose:** Deallocate all reservation objects and clear the collection.
- **Side effects:** Calls `delete` on each unique Reservation (only once per pair of ReservationEnd entries); clears `mReservationEnds`
- **Calls:** `vector::begin()`, `vector::end()`, `vector::clear()`, `delete`

### Destructor
- **Signature:** `~TextLayoutHelper()`
- **Purpose:** Clean up resources on object destruction.
- **Calls:** `removeAllReservations()`

### Constructor
- **Signature:** `TextLayoutHelper()`
- **Purpose:** Initialize (no-op; relies on member default construction)

## Control Flow Notes
This is a stateful utility class used during UI layout/rendering phases (not main game loop). It accumulates reservations incrementally as callers request space. Callers likely invoke `reserveSpaceFor()` repeatedly to place multiple UI elements, then call `removeAllReservations()` to reset for the next frame or layout pass.

## External Dependencies
- `<vector>` ΓÇö for `CollectionOfReservationEnds` storage
- `<set>` ΓÇö for temporary `multiset<Reservation*>` during overlap detection
- `<assert.h>` ΓÇö assertions on input height validity
- `TextLayoutHelper.h` ΓÇö class definition and nested struct declarations
