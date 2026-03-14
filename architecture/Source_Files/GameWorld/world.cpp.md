# Source_Files/GameWorld/world.cpp

## File Purpose
Provides low-level mathematical primitives for the game world system, including trigonometric lookup tables, coordinate transformations (translation, rotation), distance calculations, angle arithmetic, and random number generation. Serves as the foundation for spatial operations and geometry computations throughout the engine.

## Core Responsibilities
- Build and maintain precomputed trigonometry lookup tables (sine, cosine, tangent) for efficient angle-based calculations
- Perform 2D and 3D point translations and rotations relative to origins
- Convert between coordinate systems via transformation matrices
- Compute distances between points (exact 3D, exact 2D, approximate 2D heuristic)
- Normalize angles into the valid range [0, NUMBER_OF_ANGLES)
- Calculate arctangent from x/y coordinates using binary search
- Generate pseudorandom numbers (global and local LFSR-based generators)
- Compute integer square roots via bit-by-bit algorithm
- Handle coordinate overflow for long-distance calculations beyond 16-bit range

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `angle` | typedef (int16) | Angle representation; 9 bits ΓåÆ 512 angles per full rotation |
| `world_distance` | typedef (int16) | Signed 16-bit coordinate/distance value with 10 fractional bits |
| `world_point2d` | struct | 2D point with `x`, `y` as `world_distance` |
| `world_point3d` | struct | 3D point with `x`, `y`, `z` as `world_distance` |
| `long_vector2d` | struct | 32-bit intermediate vector for high-precision transforms |
| `long_vector3d` | struct | 32-bit intermediate vector for 3D high-precision transforms |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cosine_table` | int16\* | global (extern) | Precomputed cosine values for angles [0, 512) |
| `sine_table` | int16\* | global (extern) | Precomputed sine values for angles [0, 512) |
| `tangent_table` | int32\* | static | Precomputed tangent values; internal to arctangent binary search |
| `random_seed` | uint16 | static | Global LFSR state for pseudorandom number stream |
| `local_random_seed` | uint16 | static | Local LFSR state; independent random stream for per-object randomness |

## Key Functions / Methods

### build_trig_tables
- **Signature:** `void build_trig_tables(void)`
- **Purpose:** Initialize sine, cosine, and tangent lookup tables at engine startup. Allocates three tables and precomputes exact values at cardinal directions (0┬░, 90┬░, 180┬░, 270┬░).
- **Inputs:** None; uses `NUMBER_OF_ANGLES` constant.
- **Outputs/Return:** None (sets static global table pointers via malloc).
- **Side effects:** Allocates heap memory for three lookup tables; cardinal angles are hardcoded for precision.
- **Calls:** `malloc()`, `cos()`, `sin()` (C math library).
- **Notes:** Tangent is computed as `(TRIG_MAGNITUDE * sin) / cos`; avoids division by zero by storing `INT32_MIN` for 90┬░ and 270┬░.

### translate_point2d
- **Signature:** `world_point2d *translate_point2d(world_point2d *point, world_distance distance, angle theta)`
- **Purpose:** Move a 2D point by a given distance and direction (angle).
- **Inputs:** `point` (in/out), `distance`, `theta` (angle, normalized internally).
- **Outputs/Return:** Modified `point`.
- **Side effects:** In-place modification; normalizes angle.
- **Calls:** `normalize_angle()`.
- **Notes:** Uses lookup-table-scaled cosine/sine; result shifted right by `TRIG_SHIFT` (10 bits) to account for fixed-point magnitude. Comment notes ┬▒1/1024 precision margin is acceptable.

### translate_point3d
- **Signature:** `world_point3d *translate_point3d(world_point3d *point, world_distance distance, angle theta, angle phi)`
- **Purpose:** Move a 3D point by distance, azimuth (`theta`), and elevation (`phi`).
- **Inputs:** `point` (in/out), `distance`, `theta` (azimuth), `phi` (elevation/pitch).
- **Outputs/Return:** Modified `point`.
- **Side effects:** In-place modification; normalizes both angles.
- **Calls:** `normalize_angle()`.
- **Notes:** First applies `phi` (elevation) to scale horizontal distance, then applies `theta` (rotation in x-y plane) to horizontal components.

### rotate_point2d
- **Signature:** `world_point2d *rotate_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- **Purpose:** Rotate a 2D point around an origin by angle `theta`.
- **Inputs:** `point` (in/out), `origin`, `theta`.
- **Outputs/Return:** Modified `point`.
- **Side effects:** In-place modification; normalizes angle.
- **Calls:** `normalize_angle()`.
- **Notes:** Translates to origin, applies rotation matrix, translates back. Uses 32-bit intermediate values to avoid overflow.

### transform_point2d
- **Signature:** `world_point2d *transform_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- **Purpose:** Rotate a point relative to an origin and express result in origin-relative coordinates (no translation back to absolute space).
- **Inputs:** `point` (in/out), `origin`, `theta`.
- **Outputs/Return:** Modified `point` (now in local/relative space).
- **Side effects:** In-place modification.
- **Calls:** `normalize_angle()`.
- **Notes:** Similar to `rotate_point2d()` but final result is relative to origin, not absolute.

### transform_point3d
- **Signature:** `world_point3d *transform_point3d(world_point3d *point, world_point3d *origin, angle theta, angle phi)`
- **Purpose:** Rotate a 3D point in origin-relative coordinates; applies `theta` (x-y rotation) then `phi` (x-z rotation).
- **Inputs:** `point` (in/out), `origin`, `theta`, `phi`.
- **Outputs/Return:** Modified `point`.
- **Side effects:** In-place modification.
- **Calls:** None visible (manual matrix math).
- **Notes:** Two-step rotation: first x-y plane by theta, then x-z plane by phi. Skips phi rotation if `phi == 0` for efficiency.

### arctangent
- **Signature:** `angle arctangent(int32 x, int32 y)`
- **Purpose:** Compute the angle (azimuth) from a given x, y displacement using binary search in the tangent table.
- **Inputs:** `x`, `y` (32-bit signed displacements).
- **Outputs/Return:** `angle` in [0, NUMBER_OF_ANGLES).
- **Side effects:** Normalizes result via `NORMALIZE_ANGLE()` macro.
- **Calls:** `NORMALIZE_ANGLE()` macro.
- **Notes:** Reduces all quadrants to first octant, computes tangent ratio, binary-searches tangent table for closest match, maps back to original octant. Handles 0/0 gracefully (returns 0). Loren Petrich's rewrite made this long-distance friendly (accepts int32 instead of world_distance).

### isqrt
- **Signature:** `int32 isqrt(register uint32 x)`
- **Purpose:** Compute the integer (floor) square root of a 32-bit unsigned value using bit-by-bit Newton-like algorithm.
- **Inputs:** `x` (unsigned 32-bit value).
- **Outputs/Return:** Floor of sqrt(x).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Single-pass iterative algorithm: starts at `0x8000` bit, iteratively adds bits if `(r + m)┬▓ Γëñ x`. Rounds up if remainder exceeds `r`. Extensive inline documentation describes the optimization.

### distance2d / distance3d
- **Signature:** `world_distance distance2d(world_point2d *p0, world_point2d *p1)` / `world_distance distance3d(world_point3d *p0, world_point3d *p1)`
- **Purpose:** Compute exact Euclidean distance between two points using `isqrt()`.
- **Inputs:** Two points.
- **Outputs/Return:** `world_distance` (clamped to INT16_MAX if result exceeds 16-bit range).
- **Side effects:** None.
- **Calls:** `isqrt()`.
- **Notes:** Uses 32-bit delta computation to avoid overflow; 3D version sums squared deltas, 2D sums only x, y.

### guess_distance2d
- **Signature:** `world_distance guess_distance2d(world_point2d *p0, world_point2d *p1)`
- **Purpose:** Fast approximate 2D distance via heuristic (Manhattan-hybrid): `max(dx,dy) + 0.5*min(dx,dy)`.
- **Inputs:** Two 2D points.
- **Outputs/Return:** Approximate distance (faster than `isqrt()`).
- **Side effects:** None.
- **Calls:** `GUESS_HYPOTENUSE()` macro.
- **Notes:** Used when fast rough distances are acceptable; accurate within ~12% for typical cases.

### global_random / local_random
- **Signature:** `uint16 global_random(void)` / `uint16 local_random(void)`
- **Purpose:** Generate pseudorandom 16-bit values via linear-feedback shift register (LFSR) with feedback `0xb400`.
- **Inputs:** None.
- **Outputs/Return:** Next pseudorandom uint16 value; updates global/local seed.
- **Side effects:** Modifies `random_seed` or `local_random_seed`.
- **Calls:** None.
- **Notes:** Simple Galois-configuration LFSR; `global_random()` uses shared seed (all entities see same sequence), `local_random()` uses independent seed (each entity/subsystem can have own sequence if seeded separately).

### long_to_overflow_short_2d / overflow_short_to_long_2d
- **Signature:** `void long_to_overflow_short_2d(long_vector2d& LVec, world_point2d& WVec, uint16& flags)` / `void overflow_short_to_long_2d(world_point2d& WVec, uint16& flags, long_vector2d& LVec)`
- **Purpose:** Pack/unpack 32-bit long vectors into 16-bit short points by storing upper 4 bits of each axis in the upper byte of a flags word.
- **Inputs:** Long or short vector + flags word.
- **Outputs/Return:** None (in/out parameters).
- **Side effects:** Modifies WVec/LVec and flags.
- **Calls:** None.
- **Notes:** Loren Petrich's workaround for coordinate overflow; allows addressing beyond ┬▒32k by squeezing upper 4 bits (16 values) per axis into 4 flag bits each. Sign-extends upper nibbles during unpacking.

### transform_overflow_point2d
- **Signature:** `world_point2d *transform_overflow_point2d(world_point2d *point, world_point2d *origin, angle theta, uint16 *flags)`
- **Purpose:** Transform a 2D point using the overflow/flag encoding to support long-distance coordinates.
- **Inputs:** `point`, `origin`, `theta`, `flags` (in/out).
- **Outputs/Return:** Modified `point`.
- **Side effects:** Updates `flags` to encode overflow bits of result.
- **Calls:** `normalize_angle()`, `long_to_overflow_short_2d()`.
- **Notes:** Similar to `transform_point2d()` but intermediate results use 32-bit vectors and are packed/unpacked via flags.

## Control Flow Notes
This is a utility library with no inherent frame/update loop. Functions are called by other systems (rendering, entity physics, collision) as needed during game loop phases. `build_trig_tables()` is called once at engine initialization; all other functions are invoked on-demand for spatial calculations.

## External Dependencies
- **stdlib.h**: `malloc()` (table allocation).
- **math.h**: `cos()`, `sin()`, `atan()` (used in `build_trig_tables()` to compute 2╧Ç).
- **limits.h**: `INT16_MAX`, `INT32_MIN` (range constants).
- **cseries.h**: Engine infrastructure and macros (type definitions).
- **world.h**: Game world type definitions (`world_point2d`, `world_point3d`, angle constants).
