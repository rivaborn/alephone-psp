# Source_Files/ModelView/Dim3_Loader.cpp

## File Purpose
Loads and parses 3D models in Dim3 XML format for the Marathon/Aleph One game engine. Handles vertex geometry, skeletal bone hierarchies, animation frames, and sequences with bone interpolation.

## Core Responsibilities
- Parse Dim3 XML model files and populate Model3D structures
- Extract and organize vertex positions, normals, and bone-weight data
- Build bone hierarchy from parentΓÇôchild relationships and reorder bones for depth-first traversal
- Map animation frames and sequence definitions to vertex sources
- Handle multi-pass loading for file splitting scenarios
- Report parsing errors and validation failures to debug output

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `XML_Dim3DataBlock` | class | Main XML parser wrapper inheriting from `XML_Configure`; manages file buffering and error reporting |
| `BoneTagWrapper` | struct | Stores major/minor bone tags (8 chars each) for vertex-to-bone binding |
| `NameTagWrapper` | struct | Stores 32-char frame/sequence names |
| `XML_BoundingBoxParser` | class | Parses `Bound_Box` and `View_Box` elements |
| `XML_VertexParser` | class | Parses individual vertex `v` elements with position, normal, and bone tags |
| `XML_BoneParser` | class | Parses `Bone` elements with position and parent/self tags |
| `XML_TriVertexParser` | class | Parses triangle vertex references with texture UV and normals |
| `XML_FrameParser` | class | Parses `Pose` (frame) elements with name |
| `XML_FrameBoneParser` | class | Parses per-bone transforms within frames (rotation, offset) |
| `XML_SequenceParser` | class | Parses `Animation` elements |
| `XML_SeqFrameParser` | class | Parses sequence frame references with timing/sway data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DBOut` | FILE* | static | Debug output destination; NULL if disabled |
| `VertexBoneTags` | vector<BoneTagWrapper> | static | Major/minor bone bindings for each vertex |
| `BoneOwnTags` | vector<BoneTagWrapper> | static | Self-tag and parent-tag for each bone |
| `BoneIndices` | vector<size_t> | static | Remapping table from file bone order to final sorted order |
| `FrameTags` | vector<NameTagWrapper> | static | Frame names as read from file |
| `ReadFrame` | vector<Model3D_Frame> | static | Temporary frame buffer during parsing |
| `Normals` | vector<GLfloat> | static | Flat normal component storage (3 per vertex) |
| `ModelPtr` | Model3D* | static | Pointer to model being populated |
| `Dim3_ParserInited` | bool | static | Parser tree initialization flag |
| `Dim3_RootParser`, `Dim3_Parser` | XML_ElementParser | static | Root and `Model` element parsers |
| `XML_DataBlockLoader` | XML_Dim3DataBlock | static | XML parser instance |

## Key Functions / Methods

### LoadModel_Dim3
- **Signature:** `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass)`
- **Purpose:** Main entry point; loads Dim3 XML model file into a Model3D structure, optionally clearing and resetting state on first pass.
- **Inputs:** `Spec` (file reference), `Model` (target structure), `WhichPass` (LoadModelDim3_First or LoadModelDim3_Rest)
- **Outputs/Return:** `true` if positions and vertex indices populated successfully; `false` on read/parse failure.
- **Side effects:** Clears model and static caches on first pass; populates `ModelPtr`, `VertexBoneTags`, `BoneOwnTags`, `BoneIndices`, `FrameTags`, `Normals`; writes debug output if `DBOut` set; calls `Model.FindPositions_Neutral()`, `Model.BuildInverseVSIndices()`.
- **Calls:** `Spec.Open()`, `OFile.GetLength()`, `OFile.Read()`, `Dim3_SetupParseTree()`, `XML_DataBlockLoader.ParseData()`, `Model.FindPositions_Neutral()`, `Model.BuildInverseVSIndices()`.
- **Notes:** Multi-pass design allows splitting large models across files. Bone sorting is lazy-initialized. Normals copied from flat array to model structures at end.

### GetAngle
- **Signature:** `static int16 GetAngle(float InAngle)`
- **Purpose:** Convert degrees to Marathon's internal fixed-point angle units (FULL_CIRCLE = 512 units).
- **Inputs:** `InAngle` (degrees, floating-point)
- **Outputs/Return:** int16 angle in engine units, normalized to [0, FULL_CIRCLE).
- **Side effects:** None.
- **Calls:** `NORMALIZE_ANGLE` macro.
- **Notes:** Handles both positive and negative angles; uses banker's rounding (+0.5 before truncation).

### Dim3_SetupParseTree
- **Signature:** `void Dim3_SetupParseTree()`
- **Purpose:** Lazily initialize the XML element parser hierarchy linking root, model, and all child parsers.
- **Inputs:** None.
- **Outputs/Return:** Void; modifies global parser objects.
- **Side effects:** Sets `Dim3_ParserInited` to true; calls `AddChild()` on multiple parser instances to build parse tree.
- **Calls:** Multiple `AddChild()` methods on parser objects.
- **Notes:** Called once per load session. Tree structure mirrors Dim3 XML schema.

### SetDebugOutput_Dim3
- **Signature:** `void SetDebugOutput_Dim3(FILE *DebugOutput)`
- **Purpose:** Configure optional debug output stream.
- **Inputs:** FILE pointer or NULL.
- **Outputs/Return:** Void.
- **Side effects:** Sets `DBOut` global.

### XML_VertexParser::AttributesDone
- **Signature:** `bool XML_VertexParser::AttributesDone()`
- **Purpose:** Finalize a parsed vertex; append to model's vertex sources and bond/normal caches.
- **Inputs:** None (uses member `Data`, `Norm`, `BT`).
- **Outputs/Return:** `true`.
- **Side effects:** Appends to `ModelPtr->VtxSources`, `Normals`, `VertexBoneTags`.

### XML_BoneParser::AttributesDone
- **Signature:** `bool XML_BoneParser::AttributesDone()`
- **Purpose:** Finalize a parsed bone; append to model and bone-tag cache.
- **Inputs:** None (uses member `Data`, `BT`).
- **Outputs/Return:** `true`.
- **Side effects:** Appends to `ModelPtr->Bones`, `BoneOwnTags`.

### Bone sorting section (within LoadModel_Dim3)
- **Purpose:** Reorder bones into depth-first traversal order, setting Push/Pop flags for skeleton stack management.
- **Algorithm:** Build a stack-based traversal; for each unvisited bone, find one whose parent is on the stack or is a root. Assign `BoneIndices` mapping and insert Push/Pop markers.
- **Invariant:** Each bone's parent must either be in the hierarchy or treated as a virtual root.
- **Notes:** Asserts on circular references; handles detached subtrees by resetting stack.

## Control Flow Notes
- **Init phase:** `LoadModel_Dim3` called; on first pass, clears static state and sets `ModelPtr`.
- **Parse phase:** Expat library drives XML callbacks; each element type's parser accumulates data into static temporary containers.
- **Post-parse phase:** Bone reordering and normal unpacking happen in `LoadModel_Dim3` after XML parsing completes.
- **Frame/sequence phase:** Animation data accumulated into `ModelPtr->Frames` and `ModelPtr->SeqFrames` during parsing.

## External Dependencies
- **Includes:** `cseries.h` (core library), `Dim3_Loader.h` (interface), `world.h` (angle constants: FULL_CIRCLE, NORMALIZE_ANGLE), `XML_Configure.h`, `XML_ElementParser.h`
- **Expat:** XML parser library (included indirectly via XML_Configure.h)
- **Defined elsewhere:** Model3D, Model3D_Bone, Model3D_Frame, Model3D_VertexSource, Model3D_SeqFrame, FileSpecifier, OpenedFile, NONE, UNONE constants
