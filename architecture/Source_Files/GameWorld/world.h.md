# Source_Files/GameWorld/world.h

## File Purpose
Core geometry and world-coordinate system definitions for the game engine. Defines fixed-point world coordinates, angles, trigonometric constants, and data structures for points, vectors, and 3D locations. Declares coordinate transformation, distance calculation, and random number generation functions.

## Core Responsibilities
- Define fixed-point world distance units and coordinate system constants
- Provide angle/trigonometric constants and normalization utilities
- Declare point and vector data structures (2D, 3D, fixed-point, and 64-bit long variants)
- Export trigonometric lookup table globals
- Declare coordinate transformation functions (rotation, translation, combined transform)
- Declare distance calculation functions (exact and approximate variants)
- Provide random number generation interface
- Support long-integer intermediate calculations for precision in distant geometry

## Key Types / Data Structures

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|-----------|---------|
| `world_point2d` | struct | 2D world position (x, y) |
| `world_point3d` | struct | 3D world position (x, y, z) |
| `world_vector2d` | struct | 2D displacement vector (i, j) |
| `world_vector3d` | struct | 3D displacement vector (i, j, k) |
| `fixed_point3d` | struct | 3D fixed-point coordinate for high precision |
| `fixed_vector3d` | struct | 3D fixed-point vector |
| `long_point2d`, `long_point3d` | struct | 32-bit intermediate coordinates for precision |
| `long_vector2d`, `long_vector3d` | struct | 32-bit intermediate vectors |
| `world_location3d` | struct | Full 3D state: position, polygon index, yaw/pitch angles, velocity |
| `angle` | typedef | int16 angle value |
| `world_distance` | typedef | int16 fixed-point distance unit |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cosine_table` | short* | extern global | Precomputed cosine lookup values |
| `sine_table` | short* | extern global | Precomputed sine lookup values |

## Key Functions / Methods

### build_trig_tables
- Signature: `void build_trig_tables(void)`
- Purpose: Initialize cosine and sine lookup tables for trigonometric calculations
- Inputs: None
- Outputs: None
- Side effects: Populates global `cosine_table` and `sine_table`
- Calls: (defined in WORLD.C)
- Notes: Must be called during engine initialization before any trigonometric operations

### normalize_angle
- Signature: `static inline angle normalize_angle(angle theta)`
- Purpose: Wrap angle to [0, 512) range
- Inputs: theta (raw angle, any int16 value)
- Outputs: Normalized angle
- Side effects: None
- Calls: NORMALIZE_ANGLE macro (bitwise AND with NUMBER_OF_ANGLESΓêÆ1)
- Notes: Inlined for speed; uses bit-trick for O(1) wrapping

### rotate_point2d / rotate_point3d
- Signature: `world_point2d *rotate_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- Purpose: Rotate point around origin by angle
- Inputs: point, origin, rotation angle
- Outputs: Pointer to point (modified in-place)
- Side effects: Modifies input point
- Calls: Trigonometric functions (defined elsewhere)
- Notes: Returns pointer for chaining; 3D variant takes yaw and pitch angles

### translate_point2d / translate_point3d
- Signature: `world_point2d *translate_point2d(world_point2d *point, world_distance distance, angle theta)`
- Purpose: Move point by distance units in direction theta
- Inputs: point, distance, direction angle
- Outputs: Pointer to point (modified in-place)
- Side effects: Modifies input point
- Calls: Trigonometric functions
- Notes: Modifies input in-place

### transform_point2d / transform_point3d
- Signature: `world_point2d *transform_point2d(world_point2d *point, world_point2d *origin, angle theta)`
- Purpose: Combined rotation around origin followed by relative translation
- Inputs: point, origin, rotation angle
- Outputs: Pointer to point (modified in-place)
- Side effects: Modifies input point
- Calls: Trigonometric functions
- Notes: 3D version takes yaw and pitch; composite operation

### arctangent
- Signature: `angle arctangent(int32 x, int32 y)`
- Purpose: Calculate angle from 2D displacement vector; designed for long-distance calculations
- Inputs: x, y (32-bit to preserve precision for large magnitudes)
- Outputs: Angle in [0, 512)
- Side effects: Reads internal tangent lookup table
- Calls: Tangent table lookup
- Notes: LP addition (2000) for long-distance friendliness; takes int32 instead of int16

### distance2d / distance3d / guess_distance2d
- Signature: `world_distance distance2d(world_point2d *p0, world_point2d *p1)` (exact); `guess_distance2d()` (approximate)
- Purpose: Calculate Euclidean distance between two points
- Inputs: Two points
- Outputs: Distance as world_distance
- Side effects: None (distance2d calls isqrt)
- Calls: isqrt for exact distance
- Notes: `guess_distance2d` uses GUESS_HYPOTENUSE macro for O(1) approximation; `distance2d` is slower but accurate

### set_random_seed / get_random_seed / global_random / local_random
- Signature: `void set_random_seed(uint16 seed); uint16 get_random_seed(void); uint16 global_random(void); uint16 local_random(void)`
- Purpose: Initialize and query global RNG state, generate pseudo-random values
- Inputs: seed (for set_random_seed)
- Outputs: uint16 random value or current seed
- Side effects: Modifies global RNG state
- Calls: (defined in WORLD.C)
- Notes: local_random likely thread-safe or per-entity variant

### long_to_overflow_short_2d / overflow_short_to_long_2d
- Signature: `void long_to_overflow_short_2d(long_vector2d& LVec, world_point2d& WVec, uint16& flags)`
- Purpose: Bidirectional conversion between 32-bit and 16-bit representations with precision preservation
- Inputs: Source vector and flags
- Outputs: Destination vector and updated flags
- Side effects: Modifies destination and flags
- Calls: None
- Notes: LP kludge to store overflow digits (upper 4 bits of X, Y) in a flags byte; supports long-distance calculation precision

### transform_overflow_point2d
- Signature: `world_point2d *transform_overflow_point2d(world_point2d *point, world_point2d *origin, angle theta, uint16 *flags)`
- Purpose: Coordinate transform (rotate + translate) preserving overflow precision via flags
- Inputs: point, origin, angle, flags pointer
- Outputs: Pointer to point (modified in-place)
- Side effects: Modifies point and flags
- Calls: Trigonometric functions
- Notes: Specialized variant of transform_point2d for long-distance geometry

### long_to_short_2d / short_to_long_2d
- Purpose: Simple vector type conversion (truncation or promotion)
- Notes: Inline conversions; no precision preservation (unlike overflow variants)

## Control Flow Notes
This is a data-definition and declaration-only header with no control flow. Initialization dependency: `build_trig_tables()` must be called during engine startup. Random seed should be set via `set_random_seed()` at game initialization. No frame loop integration is inferable from this file.

## External Dependencies
- **Includes/typedefs used**: `int16`, `int32`, `uint16`, `uint32`, `_fixed`, `register` (defined elsewhere)
- **Global symbols defined elsewhere**: `cosine_table`, `sine_table` (WORLD.C)
- **Dependency**: `FIXED_FRACTIONAL_BITS` constant (likely fixed-point arithmetic header)
- **Function definitions**: All implementations in WORLD.C
