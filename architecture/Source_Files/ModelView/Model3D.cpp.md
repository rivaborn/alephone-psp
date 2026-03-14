# Source_Files/ModelView/Model3D.cpp

## File Purpose
Implements 3D model data storage and manipulation for skeletal animation, including vertex transformation via bone hierarchies, frame interpolation, normal computation, and bounding box management. Designed for OpenGL rendering in the Aleph One engine (Marathon source port).

## Core Responsibilities
- Apply skeletal deformations to vertices using bone hierarchies and animation frames
- Interpolate between animation frames with smooth blending
- Compute vertex and per-polygon normals with optional smoothing and vertex splitting
- Build and manage vertex-source inverse indices for efficient animation lookups
- Maintain bounding boxes and provide debug visualization
- Handle transformation matrix operations (4├ù3 matrices for 3D affine transforms)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Model3D_Transform` | struct | 3├ù4 transformation matrix (3 rows = rotation columns + translation, 4th column = offset); used for bone deformation and frame transformations |
| `FlaggedVector` | struct (local) | Temporary holder for 3D vector + validity flag; used during normal calculation to track degenerate normals |
| `Model3D_VertexSource` | struct | Vertex blend weights for two bones (Bone0, Bone1, Blend factor) |
| `Model3D_Bone` | struct | Bone position and hierarchy flags (Pop, Push for stack-based tree traversal) |
| `Model3D_Frame` | struct | Per-bone transform: offset and 3 rotation angles |
| `Model3D_SeqFrame` | struct | Extends Frame; adds Frame index for sequence animation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BoneMatrices` | `vector<Model3D_Transform>` | static | Workspace buffer for per-bone cumulative transformation matrices during frame animation |
| `BoneStack` | `vector<size_t>` | static | Workspace stack for bone hierarchy traversal during frame processing |
| `TrigNorm` | `const GLfloat` | static | Normalization constant (`1.0 / TRIG_MAGNITUDE`) for trig table lookups |

## Key Functions / Methods

### TransformPoint (inline)
- **Signature:** `void TransformPoint(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose:** Apply affine transformation (rotation + translation) to a 3D point.
- **Inputs:** source 3-vector, transformation matrix
- **Outputs/Return:** destination 3-vector (modified in-place)
- **Side effects:** None (no global state)
- **Calls:** `ScalarProd` (vector dot product)
- **Notes:** Source and dest must be different arrays; implements `Dest = T┬╖Src + offset`.

### TransformVector (inline)
- **Signature:** `void TransformVector(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose:** Apply rotation (no translation) to a vector, typically normals.
- **Inputs:** source 3-vector, transformation matrix
- **Outputs/Return:** destination 3-vector
- **Side effects:** None
- **Calls:** `ScalarProd`
- **Notes:** Ignores translation component of matrix; used for normal transformations to preserve direction.

### FindFrameTransform (static)
- **Signature:** `void FindFrameTransform(Model3D_Transform& T, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame)`
- **Purpose:** Build a 3├ù4 transformation matrix from frame data (position offset and three rotation angles), with optional interpolation between two frames.
- **Inputs:** frame data (offset and ZXY Euler angles), mix fraction, optional additional frame for blending
- **Outputs/Return:** destination transformation matrix
- **Side effects:** None
- **Calls:** `InterpolateAngle` (for each rotation axis); uses precomputed `cosine_table` and `sine_table`
- **Notes:** Applies rotations in ZXY order (Tomb Raider convention); `MixFrac` interpolates rotation angles and offset between Frame and AddlFrame.

### FindBoneTransform (static)
- **Signature:** `void FindBoneTransform(Model3D_Transform& T, Model3D_Bone& Bone, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame)`
- **Purpose:** Extend `FindFrameTransform` by incorporating bone-space positioning (bone position).
- **Inputs:** bone definition (position), frame data, mix fraction, optional additional frame
- **Outputs/Return:** destination transformation matrix
- **Side effects:** None
- **Calls:** `FindFrameTransform`, `ScalarProd`
- **Notes:** Adjusts the translation to account for bone position in model space.

### TMatMultiply (static)
- **Signature:** `void TMatMultiply(Model3D_Transform& Res, Model3D_Transform& A, Model3D_Transform& B)`
- **Purpose:** Multiply two 3├ù4 transformation matrices: `Res = A ├ù B`.
- **Inputs:** matrices A and B
- **Outputs/Return:** result matrix Res
- **Side effects:** None
- **Calls:** None
- **Notes:** Handles rotation part (3├ù3) and translation separately; necessary for composing bone hierarchies.

### Model3D::Clear
- **Signature:** `void Model3D::Clear()`
- **Purpose:** Erase all geometry, animation, and hierarchy data.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears all member vectors; recalculates bounding box
- **Calls:** `FindBoundingBox`
- **Notes:** Used when loading new models.

### Model3D::AdjustNormals
- **Signature:** `void Model3D::AdjustNormals(int NormalType, float SmoothThreshold)`
- **Purpose:** Compute or modify normals based on mode: flat (per-polygon), smooth (per-vertex averaged), or split (per-vertex per-polygon for hard edges).
- **Inputs:** normal type enum (None, Original, Reversed, ClockwiseSide, CounterclockwiseSide), smooth threshold for variance-based splitting
- **Outputs/Return:** modifies `Normals` vector and potentially `VertIndices` (vertex splitting)
- **Side effects:** Reallocates and rewrites geometry arrays; may increase vertex count via splitting
- **Calls:** `NormalizeNormal`, `objlist_copy`, `objlist_clear`, vector operations
- **Notes:** Complex algorithm: computes per-polygon normals ΓåÆ accumulates to per-vertex ΓåÆ decides splitting via variance test ΓåÆ remaps vertices and duplicates data. The most intricate function in the file.

### Model3D::FindBoundingBox
- **Signature:** `void Model3D::FindBoundingBox()`
- **Purpose:** Compute axis-aligned bounding box from current vertex positions.
- **Inputs:** None (uses `Positions` vector)
- **Outputs/Return:** sets `BoundingBox[0]` (min) and `BoundingBox[1]` (max)
- **Side effects:** Modifies member state
- **Calls:** `VecCopy`, `MIN`/`MAX` macros
- **Notes:** Called after geometry changes; handles empty position arrays.

### Model3D::RenderBoundingBox
- **Signature:** `void Model3D::RenderBoundingBox(const GLfloat *EdgeColor, const GLfloat *DiagonalColor)`
- **Purpose:** Debug-only OpenGL rendering of bounding box edges and face diagonals.
- **Inputs:** color pointers (RGB triplets; NULL = skip that set)
- **Outputs/Return:** None
- **Side effects:** OpenGL state changes and draw calls
- **Calls:** `glColor3fv`, `glVertexPointer`, `glDrawElements`
- **Notes:** Uses static index arrays for edges (24 verts) and diagonals (24 verts).

### Model3D::BuildInverseVSIndices
- **Signature:** `void Model3D::BuildInverseVSIndices()`
- **Purpose:** Precompute reverse lookup from vertex sources to vertex indices for efficient animation blending.
- **Inputs:** None (uses `VtxSrcIndices` and `VtxSources`)
- **Outputs/Return:** populates `InverseVSIndices` and `InvVSIPointers`
- **Side effects:** Allocates and fills member vectors
- **Calls:** `objlist_clear`
- **Notes:** Creates a compressed list: `InvVSIPointers[i]` and `InvVSIPointers[i+1]` bound the vertices using source i.

### Model3D::FindPositions_Neutral
- **Signature:** `bool Model3D::FindPositions_Neutral(bool UseModelTransform)`
- **Purpose:** Populate vertex positions from vertex sources without animation (identity frame).
- **Inputs:** whether to apply model's overall transform
- **Outputs/Return:** true if vertex sources were present; modifies `Positions` and `Normals`
- **Side effects:** Reallocates geometry vectors
- **Calls:** `TransformPoint`, `TransformVector`, `objlist_copy`
- **Notes:** Used for idle/neutral pose; optionally applies `TransformPos` and `TransformNorm` matrices.

### Model3D::FindPositions_Frame
- **Signature:** `bool Model3D::FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)`
- **Purpose:** Generate vertex positions by applying bone deformations from a specific animation frame (with optional blending to a second frame).
- **Inputs:** frame index, mix fraction (0.0ΓÇô1.0) for blending, additional frame index, model-transform flag
- **Outputs/Return:** true if frame index valid; modifies `Positions` and `Normals`
- **Side effects:** Allocates workspace (`BoneMatrices`, `BoneStack`); rewrites geometry
- **Calls:** `FindBoneTransform`, `TMatMultiply`, `TransformPoint`, `TransformVector`, `BuildInverseVSIndices`, vector ops
- **Notes:** Core skeletal animation logic: builds per-bone matrices ΓåÆ applies stack-based hierarchy (Pop/Push) to compute cumulative transforms ΓåÆ blends vertices across up to two bones each ΓåÆ applies optional model transform.

### Model3D::FindPositions_Sequence
- **Signature:** `bool Model3D::FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac, GLshort AddlFrameIndex)`
- **Purpose:** Apply sequence-level coordinate transformations on top of frame-based skeletal animation.
- **Inputs:** sequence index, frame within sequence, mix fraction, optional additional frame
- **Outputs/Return:** true if indices valid; modifies `Positions` and `Normals`
- **Side effects:** Reallocates geometry
- **Calls:** `NumSeqFrames`, `FindFrameTransform`, `FindPositions_Frame`, `TMatMultiply`, `TransformPoint`, `TransformVector`
- **Notes:** Retrieves a `SeqFrame` which holds a frame index and an overall offset/rotation; composes this with the bone animation via matrix multiplication.

### Model3D::NumSeqFrames
- **Signature:** `GLshort Model3D::NumSeqFrames(GLshort SeqIndex)`
- **Purpose:** Return the number of frames in a sequence.
- **Inputs:** sequence index
- **Outputs/Return:** frame count, or 0 if out of range
- **Side effects:** None
- **Calls:** None
- **Notes:** Uses `SeqFrmPointers` cumulative index array.

### Model3D_Transform::Identity
- **Signature:** `void Model3D_Transform::Identity()`
- **Purpose:** Set transformation to the identity (no rotation, no translation).
- **Inputs:** None
- **Outputs/Return:** modifies matrix in-place
- **Side effects:** None
- **Calls:** `obj_clear`
- **Notes:** Sets diagonal [0,0], [1,1], [2,2] to 1; rest to 0.

**Trivial helpers** (not detailed above): `NormalizeNormal`, `InterpolateAngle`, `BuildTrigTables`, `Clear`.

## Control Flow Notes
- **Initialization:** Constructor calls `FindBoundingBox()` and sets transforms to identity.
- **Static model:** Call `FindPositions_Neutral()` once to populate geometry.
- **Frame animation:** Call `FindPositions_Frame()` each frame, optionally with `MixFrac` for smooth crossfades between keyframes.
- **Sequence animation:** Call `FindPositions_Sequence()` for sequence-level control (e.g., walk cycles with additive transforms).
- **Normal adjustment:** Call `AdjustNormals()` after geometry is finalized; decoupled from animation.
- **Bounding box:** Updated on `Clear()`; can be manually refreshed if needed.

## External Dependencies
- **VecOps.h:** `VecCopy`, `VecAdd`, `VecSub`, `VecAddTo`, `VecScalarMultTo`, `ScalarProd` (vector math).
- **OpenGL:** `GLfloat`, `GLushort`, `GLshort` types; `glDisable`, `glColor3fv`, `glVertexPointer`, `glDrawElements`.
- **world.h:** `cosine_table`, `sine_table` (precomputed trig), `NORMALIZE_ANGLE`, `TRIG_MAGNITUDE`, `FULL_CIRCLE`, `HALF_CIRCLE`, `UNONE` (magic constant for "no index").
- **cseries.h:** `objlist_copy`, `objlist_clear`, `obj_copy`, `obj_clear`, `TEST_FLAG` (utility macros); also includes SDL and platform-specific headers.
- **Model3D.h:** Type definitions (`Model3D_VertexSource`, `Model3D_Bone`, `Model3D_Frame`, `Model3D_SeqFrame`, `Model3D_Transform`, `Model3D`).
