# Source_Files/Misc/Random.h

## File Purpose
Random-number generator struct implementing Marsaglia's algorithms. Provides multiple RNG strategies (KISS, MWC, SHR3, CONG, LFIB4, SWB) and utility functions for generating random integers and floats. Designed as an instantiable class to support independent random streams.

## Core Responsibilities
- Implement multiple competing random-number algorithms from Marsaglia
- Maintain independent state for each RNG instance (seed values and lookup tables)
- Provide high-quality 32-bit pseudo-random integers
- Generate uniform and signed-uniform floating-point values in (0,1) and (-1,1)
- Initialize lookup table for lagged-Fibonacci generators

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GM_Random | struct | State container for all Marsaglia RNG algorithms; supports multiple independent instances |

## Global / File-Static State
None.

## Key Functions / Methods

### MWC (Multiply-With-Carry)
- Signature: `uint32 MWC()`
- Purpose: Combined multiply-with-carry generator using two sub-generators (znew, wnew)
- Outputs: 32-bit pseudo-random integer
- Calls: `znew()`, `wnew()`
- Notes: Shifts znew result left 16 bits before adding wnew; one of the faster generators

### KISS (Keep It Simple Stupid)
- Signature: `uint32 KISS()`
- Purpose: Combine three independent generators (MWC + CONG + SHR3) for improved statistical properties
- Outputs: 32-bit pseudo-random integer
- Calls: `MWC()`, `CONG()`, `SHR3()`
- Notes: Recommended as primary generator; combines strength of three different algorithms

### SHR3 (Shift-Register)
- Signature: `uint32 SHR3()`
- Purpose: Shift-register generator using XOR feedback
- Outputs: 32-bit pseudo-random integer
- Notes: Operates via state feedback (jsr mutated in-place); period determined by feedback taps (shifts by 17, 13, 5)

### UNI / VNI (Uniform distributions)
- Signature: `float UNI()` / `float VNI()`
- Purpose: Convert raw 32-bit integers to floating-point ranges
- UNI outputs: float in (0,1)
- VNI outputs: float in (-1,1) using signed interpretation
- Calls: `KISS()`
- Notes: Uses magic constants for floating-point conversion; VNI casts to int32 first for sign handling

### SetTable
- Signature: `void SetTable()`
- Purpose: Initialize 256-element lookup table with pseudo-random values
- Side effects: Populates `t[256]`
- Calls: `KISS()` 256 times
- Notes: Must be called during initialization; enables lagged-Fibonacci generators (LFIB4, SWB)

### Constructor (GM_Random)
- Signature: `GM_Random() : z(...), w(...), jsr(...), jcong(...), x(0), y(0), c(0) { SetTable(); }`
- Purpose: Initialize a new RNG instance with hardcoded seed values
- Side effects: Calls `SetTable()`, initializes all state members
- Notes: All seeds are compile-time constants; no user-provided seeding in constructor

## Control Flow Notes
This is a utility class with no control flow. Game code instantiates `GM_Random` objects and calls methods on-demand to generate random numbers. No implicit frame/update cycle. Typical usage: instantiate once, call generator methods repeatedly.

## External Dependencies
- Standard `uint32`, `int32` types (platform-defined, likely from a common header)
- No external includes visible; assumes types are already defined
