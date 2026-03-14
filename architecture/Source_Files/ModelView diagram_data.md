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

## External Dependencies
- **Includes:** `cseries.h` (core library), `Dim3_Loader.h` (interface), `world.h` (angle constants: FULL_CIRCLE, NORMALIZE_ANGLE), `XML_Configure.h`, `XML_ElementParser.h`
- **Expat:** XML parser library (included indirectly via XML_Configure.h)
- **Defined elsewhere:** Model3D, Model3D_Bone, Model3D_Frame, Model3D_VertexSource, Model3D_SeqFrame, FileSpecifier, OpenedFile, NONE, UNONE constants

# Source_Files/ModelView/Dim3_Loader.h
## File Purpose
Header providing an interface for loading 3D models in Dim3 format. Supports multipass loading where the first pass initializes structures and subsequent passes populate data. Includes debug output redirection for loader diagnostics.

## Core Responsibilities
- Declare the Dim3 model loading function with multipass support
- Define pass-type constants for coordinating multipass loads
- Provide debug output configuration interface
- Abstract file I/O through FileSpecifier abstraction
- Populate Model3D structures from Dim3 binary/text format

## External Dependencies
- **Model3D.h**: Model3D structure for storing loaded geometry, bones, frames, and animation sequences
- **FileHandler.h**: FileSpecifier abstraction for cross-platform file access
- **stdio.h**: FILE type for debug output redirection

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

## External Dependencies
- **VecOps.h:** `VecCopy`, `VecAdd`, `VecSub`, `VecAddTo`, `VecScalarMultTo`, `ScalarProd` (vector math).
- **OpenGL:** `GLfloat`, `GLushort`, `GLshort` types; `glDisable`, `glColor3fv`, `glVertexPointer`, `glDrawElements`.
- **world.h:** `cosine_table`, `sine_table` (precomputed trig), `NORMALIZE_ANGLE`, `TRIG_MAGNITUDE`, `FULL_CIRCLE`, `HALF_CIRCLE`, `UNONE` (magic constant for "no index").
- **cseries.h:** `objlist_copy`, `objlist_clear`, `obj_copy`, `obj_clear`, `TEST_FLAG` (utility macros); also includes SDL and platform-specific headers.
- **Model3D.h:** Type definitions (`Model3D_VertexSource`, `Model3D_Bone`, `Model3D_Frame`, `Model3D_SeqFrame`, `Model3D_Transform`, `Model3D`).

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

## External Dependencies
- **OpenGL** (`GL/gl.h`, platform-specific includes): GLfloat, GLushort, GLshort types; platform compatibility for macOS, Windows
- **STL** `<vector>`: Container for all dynamic arrays
- **Marathon engine** (referenced): `world.h` has `build_trig_tables()` and trig function lookup tables
- **Aleph One** project: GPL-licensed fork/continuation of Marathon

# Source_Files/ModelView/ModelRenderer.cpp
## File Purpose
Implements 3D model rendering with support for multipass shading, depth-sorted polygon rendering (when Z-buffer unavailable), and shader callbacks for texturing and external lighting. Part of the Aleph One game engine.

## Core Responsibilities
- Render 3D models with multiple shader passes and optimization based on Z-buffer availability
- Compute centroid depths and depth-sort triangles for proper back-to-front rendering without Z-buffer
- Configure OpenGL state for texturing, color arrays, and external lighting per render pass
- Manage internal scratch buffers (centroid indices, sorted vertex indices, lighting colors) to avoid repeated allocation
- Delegate texture and lighting computation to external callbacks for shader flexibility

## External Dependencies
- **OpenGL** (conditional on `HAVE_OPENGL` define): vertex arrays, draw elements, state flags.
- **STL:** `<algorithm>` for `std::sort`, `<vector>` for dynamic arrays.
- **Local headers:** `"ModelRenderer.h"` (class definition), `"cseries.h"` (Aleph One platform abstractions).
- **Defined elsewhere:** `Model3D` class, `ModelRenderShader` struct, `TEST_FLAG` macro, `obj_clear()` macro.

# Source_Files/ModelView/ModelRenderer.h
## File Purpose
Defines the ModelRenderer class for rendering 3D model objects using OpenGL. Supports both Z-buffered and depth-sorted rendering modes, with multipass shader-based rendering and polygon sorting by centroid depth.

## Core Responsibilities
- Coordinate multipass rendering of 3D models with multiple shaders
- Perform depth-sorting of polygons by centroid when Z-buffer is unavailable
- Manage shader callbacks for texture and lighting operations
- Maintain persistent vertex/centroid depth arrays to avoid re-allocation across render calls
- Support separable and non-separable shader passes
- Handle external lighting colors and semitransparency flags

## External Dependencies
- `Model3D.h`: Defines 3D model structure (vertices, bones, frames, sequences, normals, texture coordinates, colors)
- **OpenGL types:** GLfloat, GLushort, GLfloat arrays (vertices, normals, colors, texture coordinates)
- **STL:** `vector<>` for dynamic arrays

# Source_Files/ModelView/QD3D_Loader.h
## File Purpose
Header for loading QuickDraw 3D (QD3D) and Quesa format 3D model files into the engine. Provides model loading and tesselation configuration for converting curved surfaces to triangle meshes.

## Core Responsibilities
- Load 3D models from QD3D/Quesa format files into `Model3D` objects
- Configure debug/status message output for the loader
- Configure tesselation parameters (subdivision control for curved surfaces)
- Reset tesselation configuration to defaults

## External Dependencies
- **`<stdio.h>`** ΓÇö `FILE` type (debug stream)
- **`Model3D.h`** ΓÇö `Model3D` struct (geometry/animation storage)
- **`FileHandler.h`** ΓÇö `FileSpecifier` class (cross-platform file abstraction)
- **Not visible:** QuickDraw 3D or Quesa library headers (implementation detail)

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

## External Dependencies
- **Notable includes:** `cseries.h` (type defs, macros), `StudioLoader.h` (header), `Packing.h` (endian-aware binary unpacking macros and templates).
- **External symbols (defined elsewhere):**
  - `OpenedFile` class (file I/O abstraction from FileHandler.h)
  - `FileSpecifier` class (file path abstraction)
  - `Model3D` class (mesh container with Positions, TxtrCoords, VertIndices arrays)
  - `StreamToValue()`, `StreamToList()` (macros/inline functions from Packing.h; resolve to little-endian variants)
  - `GLfloat` (from OpenGL)
  - Standard library: `vector<uint8>`, `fprintf()`, `assert()`

# Source_Files/ModelView/StudioLoader.h
## File Purpose
Header file declaring the interface for loading Autodesk 3D Studio Max (.3ds) model files into the engine's Model3D representation. Provides functions to load 3D models from disk and configure debug message output during loading.

## Core Responsibilities
- Declare the main entry point for 3DS model file loading
- Provide a global debug output configuration mechanism
- Abstract 3DS format parsing and file I/O from callers

## External Dependencies
- `<stdio.h>` ΓÇö for FILE type  
- `Model3D.h` ΓÇö defines Model3D struct containing positions, normals, texture coordinates, bones, frames, sequences, and bounding box  
- `FileHandler.h` ΓÇö defines FileSpecifier for cross-platform file abstraction  
- Implementation details (3DS binary format parsing) in StudioLoader.cpp (not provided)

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

## External Dependencies
- **Model3D** ΓÇö defines output structure with `Positions`, `TxtrCoords`, `Normals`, `VertIndices` vectors and `Clear()` method; uses GLfloat
- **FileSpecifier, OpenedFile** ΓÇö file handling abstractions (defined elsewhere)
- **GLfloat** ΓÇö OpenGL floating-point type
- **Standard library:** `<vector>`, `<algorithm>` (for `sort()`), `<ctype.h>`, `<stdlib.h>`, `<string.h>`, `<stdio.h>`

# Source_Files/ModelView/WavefrontLoader.h
## File Purpose
Public interface for loading Alias|Wavefront (.obj) 3D model files into the engine's Model3D format. Provides two entry points: one to load a model from a file, and another to configure debug output routing for the loader.

## Core Responsibilities
- Expose public API for Wavefront OBJ model import
- Support cross-platform file loading via FileSpecifier abstraction
- Enable runtime configuration of debug/status message output

## External Dependencies
- `<stdio.h>` ΓÇö for FILE type
- `Model3D.h` ΓÇö Model3D struct/class for storing parsed mesh data
- `FileHandler.h` ΓÇö FileSpecifier cross-platform file abstraction


