# Source_Files/ModelView/QD3D_Loader.h

## File Purpose
Header for loading QuickDraw 3D (QD3D) and Quesa format 3D model files into the engine. Provides model loading and tesselation configuration for converting curved surfaces to triangle meshes.

## Core Responsibilities
- Load 3D models from QD3D/Quesa format files into `Model3D` objects
- Configure debug/status message output for the loader
- Configure tesselation parameters (subdivision control for curved surfaces)
- Reset tesselation configuration to defaults

## Key Types / Data Structures
None defined in this file. Uses external types:
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (from FileHandler.h) | File reference abstraction |
| `Model3D` | struct (from Model3D.h) | 3D geometry storage (positions, normals, indices, animations, bones) |
| `FILE` | typedef (from stdio.h) | C standard file stream |

## Global / File-Static State
None visible; implementation manages internally:
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Debug output stream | `FILE*` | static | Destination for loader status messages |
| Tesselation mode | `bool` | static | Whether length is world-space or subdivision factor |
| Tesselation length | `float` | static | Tesselation parameter value |

## Key Functions / Methods

### LoadModel_QD3D
- **Signature:** `bool LoadModel_QD3D(FileSpecifier& Spec, Model3D& Model);`
- **Purpose:** Parse and load a QD3D/Quesa model file into a `Model3D` object.
- **Inputs:** File path/reference (`FileSpecifier`); output model container (`Model3D&`)
- **Outputs/Return:** `bool` ΓÇö true if load succeeded, false otherwise
- **Side effects:** Modifies `Model3D` (populates geometry, textures, bones, frames); file I/O
- **Calls:** Visible calls not shown in header; implementation likely uses QD3D/Quesa parsing APIs
- **Notes:** Respects tesselation and debug settings configured by other functions.

### SetDebugOutput_QD3D
- **Signature:** `void SetDebugOutput_QD3D(FILE *DebugOutput);`
- **Purpose:** Route loader status messages to a file stream (e.g., `stdout`, `stderr`, or custom file).
- **Inputs:** `FILE*` pointer; `NULL` disables debug output
- **Outputs/Return:** void
- **Side effects:** Sets global debug output stream
- **Calls:** Direct use only; no visible call chain
- **Notes:** Called before loading to capture loader progress/warnings.

### SetTesselation_QD3D
- **Signature:** `void SetTesselation_QD3D(bool IsWorldLength, float TessLength);`
- **Purpose:** Configure how curved/patch surfaces are subdivided into triangles.
- **Inputs:** 
  - `IsWorldLength` ΓÇö if `true`, `TessLength` is world units; if `false`, a fixed subdivision count
  - `TessLength` ΓÇö the length threshold or subdivision factor
- **Outputs/Return:** void
- **Side effects:** Sets global tesselation parameters; affects all subsequent `LoadModel_QD3D` calls
- **Calls:** Direct use only
- **Notes:** Higher (or more frequent) subdivision = finer, smoother mesh detail but higher polygon count.

### SetDefaultTesselation_QD3D
- **Signature:** `void SetDefaultTesselation_QD3D();`
- **Purpose:** Reset tesselation to built-in defaults.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Resets global tesselation configuration
- **Calls:** Direct use only
- **Notes:** Useful to undo custom tesselation settings.

## Control Flow Notes
Typical usage order:
1. Optionally call `SetDebugOutput_QD3D()` to enable logging
2. Optionally call `SetTesselation_QD3D()` to customize surface subdivision
3. Call `LoadModel_QD3D()` to parse a model file ΓåÆ populates `Model3D`
4. Use the populated `Model3D` for rendering, animation, or collision

Fits into asset-loading initialization phase (before rendering begins).

## External Dependencies
- **`<stdio.h>`** ΓÇö `FILE` type (debug stream)
- **`Model3D.h`** ΓÇö `Model3D` struct (geometry/animation storage)
- **`FileHandler.h`** ΓÇö `FileSpecifier` class (cross-platform file abstraction)
- **Not visible:** QuickDraw 3D or Quesa library headers (implementation detail)
