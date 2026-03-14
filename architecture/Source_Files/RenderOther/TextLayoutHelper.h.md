# Source_Files/RenderOther/TextLayoutHelper.h

## File Purpose
Defines a utility class for managing non-overlapping rectangular reservations in a 2D space. Used by the Marathon: Aleph One game engine to calculate optimal placement positions for text or UI elements without overlap.

## Core Responsibilities
- Reserve rectangular space with automatic collision avoidance
- Calculate optimal vertical placement given horizontal bounds and desired height
- Maintain a collection of active reservations tracked by endpoint coordinates
- Clear all reservations for reuse

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TextLayoutHelper | class | Main spatial layout manager |
| TextLayoutHelper::ReservationEnd | struct | Tracks horizontal position and marks start/end of a reservation boundary |
| TextLayoutHelper::Reservation | struct | Stores vertical bounds (top/bottom) of a reserved rectangle |
| CollectionOfReservationEnds | typedef (vector) | Container for all active reservation endpoints |

## Global / File-Static State
None.

## Key Functions / Methods

### reserveSpaceFor
- **Signature:** `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- **Purpose:** Find the lowest non-overlapping vertical position for a rectangle with given horizontal and height constraints.
- **Inputs:** Left edge x-coordinate, width, minimum acceptable bottom y-coordinate, height.
- **Outputs/Return:** Integer y-coordinate (bottom) of the placed rectangle.
- **Side effects:** Inserts a new ReservationEnd entry into `mReservationEnds` to track the placement.
- **Calls:** Not inferable from header.
- **Notes:** Core algorithm; implementation likely iterates through existing reservations to compute collision-free placement.

### removeAllReservations
- **Signature:** `void removeAllReservations()`
- **Purpose:** Clear all active reservations, resetting the layout state.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears `mReservationEnds` vector.
- **Calls:** Not inferable from header.

## Control Flow Notes
This is a stateful utility class (not loop-driven). Typical usage: instantiate, call `reserveSpaceFor()` one or more times to lay out rectangles, then destroy or `removeAllReservations()` to reset. Not tied to render/frame/update cycles.

## External Dependencies
- `#include <vector>` ΓÇö STL vector for dynamic reservation tracking
- `using namespace std` ΓÇö Brings std namespace into scope
