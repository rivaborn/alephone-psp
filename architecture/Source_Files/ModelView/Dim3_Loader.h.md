# Source_Files/ModelView/Dim3_Loader.h

## File Purpose
Header providing an interface for loading 3D models in Dim3 format. Supports multipass loading where the first pass initializes structures and subsequent passes populate data. Includes debug output redirection for loader diagnostics.

## Core Responsibilities
- Declare the Dim3 model loading function with multipass support
- Define pass-type constants for coordinating multipass loads
- Provide debug output configuration interface
- Abstract file I/O through FileSpecifier abstraction
- Populate Model3D structures from Dim3 binary/text format

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| LoadModelDim3_First | enum constant | Signals first pass: initializes loader state and model structures |
| LoadModelDim3_Rest | enum constant | Signals subsequent passes: populates model data |

## Global / File-Static State
None visible in header; implementation likely maintains file-scope state for:
- Current debug output FILE pointer (set via `SetDebugOutput_Dim3`)
- Multipass loader context (implicit in the WhichPass parameter design)

## Key Functions / Methods

### LoadModel_Dim3
- **Signature:** `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass)`
- **Purpose:** Load a Dim3 model file into a Model3D structure, supporting multipass initialization
- **Inputs:**
  - `Spec`: FileSpecifier referencing the Dim3 file to load
  - `Model`: Model3D reference to populate with loaded data
  - `WhichPass`: Pass indicator (LoadModelDim3_First or LoadModelDim3_Rest)
- **Outputs/Return:** `bool` ΓÇô success (true) or failure (false)
- **Side effects:** Modifies Model3D structure; may emit debug output to configured FILE pointer
- **Calls:** Not visible; likely calls FileHandler and Model3D methods
- **Notes:** Multipass design suggests first pass allocates/initializes, subsequent passes fill buffers. Implementation must handle partial model state between passes.

### SetDebugOutput_Dim3
- **Signature:** `void SetDebugOutput_Dim3(FILE *DebugOutput)`
- **Purpose:** Configure where loader diagnostics and status messages are printed
- **Inputs:** `DebugOutput` ΓÇô FILE pointer (stdout, stderr, log file, or NULL to disable)
- **Outputs/Return:** void
- **Side effects:** Updates global/static FILE pointer used by LoadModel_Dim3
- **Calls:** Not visible; likely stores pointer for later use
- **Notes:** Called before loading to configure output; NULL pointer likely disables debug output

## Control Flow Notes
This is an **initialization/loading** interface. The multipass design suggests a parse-then-populate pattern:
1. **First pass** (`LoadModelDim3_First`): Parse file structure, allocate Model3D arrays (positions, normals, bones, frames, etc.)
2. **Subsequent passes** (`LoadModelDim3_Rest`): Read and populate vertex data, animation frames, texture coordinates

Debug output is configured globally and used throughout both passes.

## External Dependencies
- **Model3D.h**: Model3D structure for storing loaded geometry, bones, frames, and animation sequences
- **FileHandler.h**: FileSpecifier abstraction for cross-platform file access
- **stdio.h**: FILE type for debug output redirection
