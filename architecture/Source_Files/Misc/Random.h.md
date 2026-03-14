# Source_Files/Misc/Random.h

## File Purpose
Implements a Multiply-With-Carry (MWC) random number generator using algorithms designed by Dr. George Marsaglia. Provides a reusable `GM_Random` struct with multiple PRNG algorithms and floating-point random value generation for game logic.

## Core Responsibilities
- Maintain internal state for multiple PRNG algorithms (z, w, jsr, jcong, lookup table t[256])
- Provide multiple PRNG strategies: KISS, MWC, SHR3, CONG, LFIB4, SWB
- Generate random 32-bit integers and normalized floats in ranges (0,1) and (-1,1)
- Initialize and reseed the generator with deterministic seeding via `SetTable()`
- Ensure independent instances can coexist (per-instance state, no global generators)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GM_Random` | struct | Encapsulates state and methods for multiple combined PRNG algorithms |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor (`GM_Random()`)
- **Signature:** `GM_Random()`
- **Purpose:** Initialize a new random generator instance with hardcoded seed values
- **Inputs:** None
- **Outputs/Return:** N/A (constructor)
- **Side effects:** Calls `SetTable()` to populate the 256-element lookup table with seeded values
- **Calls:** `SetTable()`, `KISS()`
- **Notes:** Uses fixed initial seeds (z=362436069, w=521288629, jsr=123456789, jcong=380116160); deterministic across runs

### `KISS()`
- **Signature:** `uint32 KISS()`
- **Purpose:** Combine multiple algorithms (MWC Γèò CONG + SHR3) into a high-quality 32-bit random integer
- **Inputs:** None (uses instance state)
- **Outputs/Return:** 32-bit unsigned integer
- **Side effects:** Updates internal state (z, w, jsr, jcong)
- **Calls:** `MWC()`, `CONG()`, `SHR3()`
- **Notes:** Primary recommended generator; good statistical properties, period ~2^123

### `UNI()`
- **Signature:** `float UNI()`
- **Purpose:** Generate normalized float in range (0,1)
- **Inputs:** None (uses KISS() internally)
- **Outputs/Return:** Float in (0,1)
- **Side effects:** Updates PRNG state via KISS()
- **Calls:** `KISS()`
- **Notes:** Multiplier 2.328306e-10 Γëê 1/2^32

### `VNI()`
- **Signature:** `float VNI()`
- **Purpose:** Generate normalized float in range (-1,1)
- **Inputs:** None (uses KISS() internally)
- **Outputs/Return:** Float in (-1,1)
- **Side effects:** Updates PRNG state via KISS()
- **Calls:** `KISS()`
- **Notes:** Multiplier 4.656613e-10 Γëê 2/2^32; interprets KISS() result as signed int32

### `SetTable()`
- **Signature:** `void SetTable()`
- **Purpose:** Populate the 256-element lookup table with seeded random values
- **Inputs:** None (uses KISS())
- **Outputs/Return:** None
- **Side effects:** Fills `t[256]` array; updates PRNG state
- **Calls:** `KISS()`
- **Notes:** Called once during construction; enables LFIB4 and SWB algorithms

### Inline helper methods
- **`znew()`, `wnew()`, `MWC()`, `SHR3()`, `CONG()`, `LFIB4()`, `SWB()`** ΓÇö individual algorithm implementations; fast inline generators providing different statistical trade-offs

## Control Flow Notes
This is a stateless utility struct (no frame/update cycle). Instantiate once per logical entity needing independent random sequences (e.g., per-enemy AI, per-level), then call `UNI()`, `VNI()`, or `KISS()` during game logic updates whenever random values are needed. No explicit shutdown or cleanup.

## External Dependencies
- **Standard types:** `uint32`, `int32`, `unsigned char`, `float` (defined elsewhere, likely platform headers)
- No game engine or external library dependencies
