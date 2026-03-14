# Source_Files/ModelView/Model3D.h

## File Purpose
Defines OpenGL-friendly data structures for 3D model storage with skeletal animation support. Provides containers for vertices, bones, animation frames, and sequences designed to work with the Marathon engine's skeletal animation system and trig-lookup tables.

## Core Responsibilities
- Store vertex geometry (positions, texture coordinates, normals, colors) with parallel arrays
- Define skeletal bone hierarchy and bone transformations for animation
- Manage animation frames and sequences with smooth crossfading
- Support vertex source system for bone deformation with blending
- Calculate and render bounding boxes for debugging
- Compute and normalize surface normals with various algorithms
- Provide position lookup for neutral pose, keyframe animation, and sequences

## Key Types / Data Structures

| Name | Kind (struct) | Purpose |
|------|---------------|---------|
| `Model3D_VertexSource` | struct | Single vertex with bone indices (0ΓÇô2) and blend weight for skeletal deformation |
| `Model3D_Bone` | struct | Bone definition: global position, traversal flags (Push/Pop) for tree structure |
| `Model3D_Frame` | struct | Per-bone transform: offset and rotation angles (3 for Marathon's trig tables) |
| `Model3D_SeqFrame` | struct | Frame variant for sequences; adds frame ID index |
| `Model3D_Transform` | struct | 3├ù4 transformation matrix for position/normal remapping |
| `Model3D` | struct | Main container: all vertex data, bones, frames, sequences, and metadata |

## Global / File-Static State
None.

## Key Functions / Methods

### FindBoundingBox
- Signature: `void FindBoundingBox()`
- Purpose: Calculate axis-aligned bounding box from model position data
- Inputs: None (uses internal `Positions` array)
- Outputs/Return: Stores result in `BoundingBox[2][3]` (min/max corners)
- Side effects: Writes to `BoundingBox` member
- Calls: (not inferable from header)
- Notes: Called automatically in constructor; updates whenever geometry changes

### RenderBoundingBox
- Signature: `void RenderBoundingBox(const GLfloat *EdgeColor, const GLfloat *DiagonalColor)`
- Purpose: Render wireframe bounding box for debugging
- Inputs: Two color pointers (nullptr to skip rendering that component)
- Outputs/Return: None
- Side effects: OpenGL draw calls
- Calls: (not inferable from header)
- Notes: Edges and diagonals can be rendered in different colors; null pointers disable rendering

### AdjustNormals
- Signature: `void AdjustNormals(int NormalType, float SmoothThreshold = 0.5)`
- Purpose: Process normals via one of 5 strategies (None, Original, Reversed, Clockwise/Counterclockwise side)
- Inputs: `NormalType` enum (0ΓÇô4), `SmoothThreshold` for vertex averaging (default 0.5)
- Outputs/Return: Updates `Normals` array
- Side effects: Rewrites/reorganizes normal data; may split vertices
- Calls: (not inferable from header)
- Notes: Enum values: `None` (deletes), `Original` (as-is), `Reversed` (flip), `ClockwiseSide`, `CounterclockwiseSide`

### Clear
- Signature: `void Clear()`
- Purpose: Erase all model data
- Inputs: None
- Outputs/Return: None
- Side effects: Clears all vectors (Positions, Bones, Frames, etc.)
- Calls: (not inferable from header)

### BuildTrigTables (static)
- Signature: `static void BuildTrigTables()`
- Purpose: Build trig lookup tables for Marathon engine's transform computations
- Inputs: None
- Outputs/Return: None
- Side effects: Global or class-level state (trig tables)
- Notes: Only needed if `build_trig_tables()` from world.h was not called elsewhere

### FindPositions_Neutral
- Signature: `bool FindPositions_Neutral(bool UseModelTransform)`
- Purpose: Compute vertex positions at the model's neutral (rest) pose
- Inputs: `UseModelTransform` (apply overall transform only for animated models)
- Outputs/Return: `true` if vertex-source data was used (skeletal animation)
- Side effects: Writes to `Positions` array
- Notes: Called when no animation frame is specified; uses only `VtxSources` if present

### FindPositions_Frame
- Signature: `bool FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0)`
- Purpose: Compute vertex positions for a keyframe with optional crossfade to a second frame
- Inputs: `FrameIndex`, `MixFrac` (blend 0ΓÇô1), optional `AddlFrameIndex` for smooth transition
- Outputs/Return: `true` if indices were in range
- Side effects: Writes to `Positions` array

### FindPositions_Sequence
- Signature: `bool FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0)`
- Purpose: Compute vertex positions for a frame within a named sequence
- Inputs: `SeqIndex` (sequence ID), `FrameIndex`, `MixFrac`, optional second frame
- Outputs/Return: `true` if indices were in range
- Side effects: Writes to `Positions` array

### NumSeqFrames
- Signature: `GLshort NumSeqFrames(GLshort SeqIndex)`
- Purpose: Get the number of frames in a sequence
- Inputs: Sequence index
- Outputs/Return: Frame count (0 if out of range)

**Notes on trivial accessors:** `PosBase()`, `TCBase()`, `NormBase()`, `ColBase()`, `VtxSIBase()`, `VtxSrcBase()`, `NormSrcBase()`, `InverseVIBase()`, `InvVSIPtrBase()`, `BoneBase()`, `VIBase()`, `FrameBase()`, `SeqFrmBase()`, `SFPtrBase()` are inline getters returning pointers to vector backing storage for OpenGL submission.

## Control Flow Notes
This is a data container with no inherent frame loop. Animation is **pull-based**: external code calls `FindPositions_*()` methods at each frame with time indices and blend factors to compute deformed vertex positions. The structure supports:
- Static (non-animated) models via `Positions` array only
- Skeletal animation via `Bones`, `Frames`, `VtxSources` (bone weights)
- Sequence playback with named animation clips via `SeqFrames` and `SeqFrmPointers`
- Smooth interpolation between keyframes via `MixFrac` parameter

## External Dependencies
- **OpenGL** (`GL/gl.h`, platform-specific includes): GLfloat, GLushort, GLshort types; platform compatibility for macOS, Windows
- **STL** `<vector>`: Container for all dynamic arrays
- **Marathon engine** (referenced): `world.h` has `build_trig_tables()` and trig function lookup tables
- **Aleph One** project: GPL-licensed fork/continuation of Marathon
