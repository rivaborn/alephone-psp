# Source_Files/ModelView/StudioLoader.cpp

## File Purpose
Loads 3D Studio Max model files (.3DS format) into the game engine. Implements chunk-based binary format parsing with hierarchical container navigation, extracting vertex positions, texture coordinates, and face indices into a Model3D structure for rendering.

## Core Responsibilities
- Parse binary 3DS file format with chunk headers and nested container structure
- Implement hierarchical chunk navigation (Master ΓåÆ Editor ΓåÆ Object ΓåÆ Trimesh)
- Extract and decode vertex positions, texture coordinates, and face indices
- Buffer and process raw chunk data into appropriate OpenGL-compatible formats
- Provide optional debug output for file loading diagnostics
- Handle little-endian binary stream parsing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ChunkHeaderData` | struct | Represents a 3DS chunk: 16-bit ID and 32-bit size |
| MASTER, EDITOR, OBJECT, TRIMESH, VERTICES, TXTR_COORDS, FACE_DATA | const uint16 | Chunk type identifiers (0x4d4d, 0x3d3d, 0x4000, 0x4100, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ChunkBuffer` | vector<uint8> | static (file) | Reusable buffer for reading chunk data payloads |
| `DBOut` | FILE* | static (file) | Destination for debug messages; NULL if disabled |
| `ModelPtr` | Model3D* | static (file) | Pointer to the output model being populated during load |

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main entry point; opens a 3DS file and populates a Model3D with mesh data.
- **Inputs:** FileSpecifier with file path; reference to Model3D to populate.
- **Outputs/Return:** `true` if load succeeded and model has positions and vertex indices; `false` on error.
- **Side effects:** Clears the Model3D, reads and parses entire file, populates ModelPtr (global).
- **Calls:** `Spec.Open()`, `ReadChunkHeader()`, `ReadContainer()` ΓåÆ chain of hierarchical readers.
- **Notes:** Returns false if file does not start with MASTER chunk (0x4d4d) or if final model is empty. Logs to DBOut if set.

### ReadChunkHeader
- **Signature:** `bool ReadChunkHeader(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Reads a 6-byte chunk header (2-byte ID + 4-byte size in little-endian).
- **Inputs:** Opened file; reference to ChunkHeaderData struct.
- **Outputs/Return:** Populates ChunkHeader on success; `false` on read error.
- **Side effects:** Advances file position by 6 bytes.
- **Calls:** `OFile.Read()`, `StreamToValue()` (from Packing.h).
- **Notes:** Assumes little-endian byte order (set via PACKED_DATA_IS_LITTLE_ENDIAN).

### LoadChunk
- **Signature:** `bool LoadChunk(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Reads a complete chunk's payload into ChunkBuffer.
- **Inputs:** Opened file; chunk header describing size.
- **Outputs/Return:** `true` on success; `false` on read error.
- **Side effects:** Resizes ChunkBuffer and populates it with payload data; advances file position.
- **Calls:** `OFile.Read()`, `SetChunkBufferSize()`.
- **Notes:** Payload size = chunk size minus 6-byte header.

### SkipChunk
- **Signature:** `bool SkipChunk(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Skips over an unknown or unhandled chunk by seeking past it.
- **Inputs:** Opened file; chunk header describing size.
- **Outputs/Return:** `true` if seek succeeded; `false` on error.
- **Side effects:** Advances file position by payload size.
- **Calls:** `OFile.GetPosition()`, `OFile.SetPosition()`.

### ReadContainer, ReadMaster, ReadEditor, ReadObject, ReadTrimesh, ReadFaceData
- **Signature (template):** `bool Read*(OpenedFile& OFile, long ParentChunkEnd)`
- **Purpose:** Recursively parse nested chunk hierarchy. Each reader loops through child chunks until reaching the parent boundary, dispatching to appropriate sub-readers or skipping unknown chunks.
- **Inputs:** Opened file; end position of parent chunk.
- **Outputs/Return:** `true` on success (reached boundary cleanly); `false` on read error or position overrun.
- **Side effects:** Advance file position; ReadFaceData and ReadTrimesh populate ModelPtrΓåöPositions, TxtrCoords, VertIndices.
- **Calls:** Recursively call each other or helper functions.
- **Notes (ReadFaceData):** Reads face count, then 4├ùuint16 entries per face (3 vertex indices + flags). Stores vertex indices in `ModelPtr->VertIndices`.

### LoadVertices
- **Signature:** `void LoadVertices()`
- **Purpose:** Extracts vertex count and positions (3 floats each) from ChunkBuffer into ModelPtrΓåÆPositions.
- **Inputs:** None (reads from ChunkBuffer, writes via ModelPtr).
- **Outputs/Return:** None.
- **Side effects:** Resizes and populates `ModelPtr->Positions`.
- **Calls:** `LoadFloats()`.

### LoadTextureCoordinates
- **Signature:** `void LoadTextureCoordinates()`
- **Purpose:** Extracts texture coordinate count and coordinates (2 floats each) from ChunkBuffer into ModelPtrΓåÆTxtrCoords.
- **Inputs:** None (reads from ChunkBuffer, writes via ModelPtr).
- **Outputs/Return:** None.
- **Side effects:** Resizes and populates `ModelPtr->TxtrCoords`.
- **Calls:** `LoadFloats()`.

### LoadFloats
- **Signature:** `void LoadFloats(int NVals, uint8 *Stream, GLfloat *Floats)`
- **Purpose:** Interprets a binary stream of 4-byte integers as IEEE 754 floating-point values and copies them to destination.
- **Inputs:** Count of floats; byte stream; destination float array.
- **Outputs/Return:** Populates Floats array.
- **Side effects:** Advances Stream pointer; byte-level memory copy (no endian conversion, assumes stream is already in target byte order).
- **Calls:** `StreamToValue()`.
- **Notes:** Works only on platforms where GLfloat is 4-byte IEEE 754 (asserted at runtime). Direct byte copying used to avoid floating-point interpretation during binary unpacking.

### SetDebugOutput_Studio
- **Signature:** `void SetDebugOutput_Studio(FILE *DebugOutput)`
- **Purpose:** Configures the debug output destination (FILE stream).
- **Inputs:** FILE pointer (or NULL to disable).
- **Outputs/Return:** None.
- **Side effects:** Updates static DBOut; subsequent functions log to this stream if non-NULL.

## Control Flow Notes
**Initialization:** LoadModel_Studio opens the file, clears the Model3D, and calls ReadMaster via ReadContainer.

**Chunk Hierarchy:** 
- ReadMaster loops through top-level chunks, dispatching EDITOR to ReadEditor, skipping others.
- ReadEditor loops through OBJECT chunks, dispatching to ReadObject.
- ReadObject (after reading object name) loops through TRIMESH chunks, dispatching to ReadTrimesh.
- ReadTrimesh loops through VERTICES, TXTR_COORDS, FACE_DATA chunks. Leaf chunks (VERTICES, TXTR_COORDS) are loaded into ChunkBuffer and immediately processed (LoadVertices/LoadTextureCoordinates). FACE_DATA is read as a container.
- ReadFaceData reads face count and indices directly into ModelPtrΓåÆVertIndices, then skips any remaining sub-chunks.

**Each container reader:** Tracks current file position against ParentChunkEnd to detect overruns and stops cleanly when reaching the boundary. If position overshoots, returns false with error logged.

## External Dependencies
- **Notable includes:** `cseries.h` (type defs, macros), `StudioLoader.h` (header), `Packing.h` (endian-aware binary unpacking macros and templates).
- **External symbols (defined elsewhere):**
  - `OpenedFile` class (file I/O abstraction from FileHandler.h)
  - `FileSpecifier` class (file path abstraction)
  - `Model3D` class (mesh container with Positions, TxtrCoords, VertIndices arrays)
  - `StreamToValue()`, `StreamToList()` (macros/inline functions from Packing.h; resolve to little-endian variants)
  - `GLfloat` (from OpenGL)
  - Standard library: `vector<uint8>`, `fprintf()`, `assert()`
