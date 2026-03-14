# Source_Files/GameWorld/editor.h

## File Purpose
Header file defining editor-related constants, version macros, and data structures for the Marathon map editor. Specifies version compatibility across Marathon editions and establishes bounds for map geometry and patrol paths.

## Core Responsibilities
- Define data version constants for Marathon One, Two, and Infinity editions
- Specify valid coordinate and height ranges for map geometry
- Define data structure for map metadata storage
- Define data structure for guard patrol path control points and flags
- Establish compile-time constraints on map complexity

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `map_index_data` | struct | Stores map metadata: level name, flags, and version info |
| `saved_path` | struct | Stores guard patrol path: control points, polygon indices, and behavior flags |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This is a header-only definitions file included at compile time by other modules that need version compatibility checks and access to editor data structures.

## External Dependencies
- `SHORT_MIN`, `SHORT_MAX` ΓÇö bounds constants (inferred from standard library or platform headers)
- `WORLD_ONE` ΓÇö world unit scale constant (defined elsewhere)
- `LEVEL_NAME_LENGTH` ΓÇö string buffer size (defined elsewhere)
- `world_point2d` ΓÇö 2D coordinate type (defined elsewhere)

**Notes:**
- Version constants (`MARATHON_ONE_DATA_VERSION` = 0, 1, 2) suggest multi-version format compatibility.
- Height bounds allow ┬▒8 world units with 1-unit minimum ceiling clearance (`MINIMUM_CEILING_HEIGHT`).
- `INVALID_HEIGHT` sentinel value below minimum floor for validation.
- Guard path struct supports up to 20 control points per path with per-point polygon association.
- `MAX_LINES_PER_VERTEX` (15) constrains polygon complexity for editor validation.
