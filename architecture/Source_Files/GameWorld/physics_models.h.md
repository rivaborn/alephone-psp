# Source_Files/GameWorld/physics_models.h

## File Purpose
Defines physics model types and constants for character movement (walking/running) in the Aleph One game engine. Provides a struct to encapsulate all physics parameters and pre-initialized constant sets for different locomotion modes.

## Core Responsibilities
- Enumerate available physics models (walking, running)
- Define the `physics_constants` struct containing all character dynamics parameters (velocity limits, accelerations, angular motion, dimensions)
- Provide hardcoded initial physics constants for each model type
- Declare serialization/deserialization functions for physics constants (pack/unpack)
- Declare physics system initialization function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `physics_constants` | struct | Encapsulates all physics parameters for a movement model: velocity caps, accelerations, gravity, angular motion, stepping, and character dimensions |
| (unnamed) | enum | Lists physics model IDs: `_model_game_walking`, `_model_game_running`, `NUMBER_OF_PHYSICS_MODELS` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `physics_models` | `static struct physics_constants[NUMBER_OF_PHYSICS_MODELS]` | static | Runtime-modifiable physics constants for each model (mutable copy) |
| `original_physics_models` | `const struct physics_constants[NUMBER_OF_PHYSICS_MODELS]` | global const | Hardcoded defaults for walking and running models |

## Key Functions / Methods
### unpack_physics_constants
- Signature: `uint8 *unpack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Deserialize physics constants from a byte stream (likely from save files or network data)
- Inputs: byte stream pointer, destination struct array, count of objects
- Outputs/Return: updated stream pointer (for chaining reads)
- Side effects: modifies Objects array
- Calls: (Not inferable from header; implementation elsewhere)
- Notes: Typical for "1-2-3 Converter" utility mentioned in comment

### pack_physics_constants
- Signature: `uint8 *pack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)`
- Purpose: Serialize physics constants to a byte stream
- Inputs: byte stream pointer, source struct array, count of objects
- Outputs/Return: updated stream pointer
- Side effects: writes to Stream
- Calls: (Not inferable from header; implementation elsewhere)

### init_physics_constants
- Signature: `void init_physics_constants()`
- Purpose: Initialize the runtime `physics_models` array (likely copies from `original_physics_models`)
- Inputs: none
- Outputs/Return: none
- Side effects: populates static `physics_models` array
- Calls: (Not inferable from header; implementation elsewhere)

## Control Flow Notes
This is **initialization/configuration code**. Called during engine startup (`init_physics_constants`), likely before character simulation begins. The `physics_models` array is probably read during character update/movement phases to drive velocity and acceleration calculations.

## External Dependencies
- `_fixed`: Fixed-point number type (defined elsewhere; used for physics precision)
- `FIXED_ONE`, `QUARTER_CIRCLE`: Macro constants (defined elsewhere; represent fixed-point 1.0 and ╧Ç/2)
- `uint8`: Standard integer type
- `size_t`: Standard size type

**Notes:** Comment references "1-2-3 Converter," suggesting legacy serialization support. File header notes 1991ΓÇô2001+ Bungie Studios and Aleph One developers; GPL 2.0 licensed.
