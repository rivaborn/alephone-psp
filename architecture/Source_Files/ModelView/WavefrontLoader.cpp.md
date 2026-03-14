# Source_Files/ModelView/WavefrontLoader.cpp

## File Purpose
Alias|Wavefront .obj file parser and loader for the Aleph One game engine. Reads 3D model geometry (positions, texture coordinates, normals) from .obj files and populates a Model3D object for rendering. Handles vertex deduplication and polygon-to-triangle conversion.

## Core Responsibilities
- Parse Wavefront .obj file format line-by-line with continuation support
- Extract and buffer vertex positions (`v`), texture coordinates (`vt`), and normals (`vn`)
- Parse face definitions (`f`) with vertex index sets and track polygon sizes
- Convert Wavefront's 1-based and negative indexing conventions to 0-based internal indices
- Deduplicate vertices by identifying unique position/texcoord/normal combinations
- Validate index ranges and data presence across all vertex sets
- Decompose polygons into triangles using fan triangulation
- Emit debug messages for logging, warnings, and errors

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `IndexedVertListCompare` | struct | Comparator for STL `sort()` to order vertex index sets; enables deduplication by finding unique entries |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DBOut` | `FILE*` | static | Debug output stream destination; set via `SetDebugOutput_Wavefront()` |
| `InputLine` | `vector<char>` | static | Dynamic line buffer (initial capacity 64) for parsing file lines with backslash continuations |

## Key Functions / Methods

### SetDebugOutput_Wavefront
- **Signature:** `void SetDebugOutput_Wavefront(FILE *DebugOutput)`
- **Purpose:** Set the destination stream for debug/warning/error messages
- **Inputs:** `DebugOutput` ΓÇö FILE pointer (or NULL to disable debug output)
- **Outputs/Return:** None
- **Side effects:** Updates static `DBOut`

### LoadModel_Wavefront
- **Signature:** `bool LoadModel_Wavefront(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main entry point; loads a Wavefront .obj file and populates a Model3D object
- **Inputs:** `Spec` ΓÇö file specifier; `Model` ΓÇö output model (will be cleared)
- **Outputs/Return:** `true` on success, `false` on error (no polygons, index out-of-range, etc.)
- **Side effects:** Modifies `Model` (calls `Clear()`, populates `Positions`, `TxtrCoords`, `Normals`, `VertIndices`); opens and reads file; emits debug messages
- **Calls:** `Model.Clear()`, `Spec.Open()`, `Spec.GetName()`, `OFile.Read()`, `CompareToKeyword()`, `GetVertIndxSet()`, `GetVertIndx()`, `sort()`, `fprintf()`
- **Notes:**  
  - Handles line continuations (`\` at line end)  
  - Validates presence and range of position/texcoord/normal indices  
  - Deduplicates vertices before populating Model  
  - Returns false on any critical error (no polygons, out-of-range indices, missing positions)

### CompareToKeyword
- **Signature:** `char *CompareToKeyword(const char *Keyword)`
- **Purpose:** Match keyword at the start of `InputLine` and return pointer to rest of line if found
- **Inputs:** `Keyword` ΓÇö null-terminated keyword string to match (e.g., `"v"`, `"f"`)
- **Outputs/Return:** Pointer to character after keyword and any trailing whitespace; NULL if keyword not matched
- **Side effects:** None
- **Calls:** `strlen()`
- **Notes:** Accepts keyword only if followed by whitespace or end-of-line; rejects embedded matches

### GetVertIndxSet
- **Signature:** `char *GetVertIndxSet(char *Buffer, short& Presence, short& PosIndx, short& TCIndx, short& NormIndx)`
- **Purpose:** Parse a single vertex index set from a face definition (e.g., `"1/2/3"`)
- **Inputs:** `Buffer` ΓÇö pointer to position/texcoord/normal indices; output-by-reference parameters for presence flags and indices
- **Outputs/Return:** Pointer to next token in buffer; NULL if no more tokens; sets `Presence` bitmask and index values
- **Side effects:** Modifies `Buffer` position
- **Calls:** `GetVertIndx()`
- **Notes:** Handles all three possible indices (position always, texcoord and normal optional); sets presence flags for each found index

### GetVertIndx
- **Signature:** `char *GetVertIndx(char *Buffer, bool& WasFound, short& Val, bool& HitEnd)`
- **Purpose:** Parse a single integer index value (e.g., `"1"`) from the buffer
- **Inputs:** `Buffer` ΓÇö pointer to digits; output-by-reference parameters for parse result
- **Outputs/Return:** Pointer past parsed index or `/` delimiter; sets `WasFound` (true if index was parsed), `Val` (the integer), and `HitEnd` (true if whitespace or end-of-line hit)
- **Side effects:** Modifies buffer position
- **Calls:** `sscanf()`
- **Notes:** Stops at `/` (index separator), whitespace, or end-of-string; index buffer limited to 63 characters

## Control Flow Notes
1. **Initialization:** `LoadModel_Wavefront()` clears the output model and opens the file.
2. **Line reading loop:** Reads file byte-by-byte, handling line continuations (`\` at EOL) and building up lines in `InputLine`.
3. **Keyword dispatch:** Each non-comment, non-empty line is checked against keywords (`v`, `vt`, `vn`, `f`); other keywords are ignored.
4. **Geometry buffering:** Vertex positions, texture coordinates, and normals are accumulated into temporary vectors; face indices are processed immediately into `VertIndxSets` and `PolygonSizes`.
5. **Validation:** All indices are validated for range and presence; polygons with size < 3 are skipped.
6. **Deduplication:** A sorted scan of vertex index sets identifies unique combinations and maps original indices to deduplicated indices.
7. **Population:** Unique vertices are copied into the Model3D object.
8. **Triangulation:** Polygons are converted to triangles via fan decomposition and indices stored in `VertIndices`.

## External Dependencies
- **Model3D** ΓÇö defines output structure with `Positions`, `TxtrCoords`, `Normals`, `VertIndices` vectors and `Clear()` method; uses GLfloat
- **FileSpecifier, OpenedFile** ΓÇö file handling abstractions (defined elsewhere)
- **GLfloat** ΓÇö OpenGL floating-point type
- **Standard library:** `<vector>`, `<algorithm>` (for `sort()`), `<ctype.h>`, `<stdlib.h>`, `<string.h>`, `<stdio.h>`
