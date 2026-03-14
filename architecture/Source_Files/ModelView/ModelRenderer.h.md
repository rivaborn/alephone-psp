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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ModelRenderShader | struct | Holds rendering configuration: flags, texture callback, and lighting callback with associated data pointers |
| IndexedCentroidDepth | struct | Pairs polygon centroid depth with vertex index; sortable for back-to-front rendering |
| ModelRenderer | class | Main orchestrator for 3D model rendering with depth-sorting and multipass support |

## Global / File-Static State
None.

## Key Functions / Methods

### Render
- **Signature:** `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer);`
- **Purpose:** Perform the actual rendering of a 3D model to the OpenGL framebuffer.
- **Inputs:** 
  - `Model`: 3D model data (geometry, bones, frames, sequences)
  - `Shaders`: Array of shader configurations for multipass rendering
  - `NumShaders`: Total shaders to apply
  - `NumSeparableShaders`: Count of shaders that can be rendered independently when Z-buffer present (always the first shaders; all-or-nothing)
  - `Use_Z_Buffer`: If true, separate shaders may render independently; if false, all polygons are depth-sorted
- **Outputs/Return:** void (renders to bound OpenGL context)
- **Side effects:** Modifies OpenGL state; populates IndexedCentroidDepths, SortedVertIndices, ExtLightColors
- **Calls:** SetupRenderPass (visible)
- **Notes:** Without Z-buffer, polygons are sorted by centroid depth using ViewDirection; semitransparent shaders are always non-separable

### SetupRenderPass
- **Signature:** `void SetupRenderPass(Model3D& Model, ModelRenderShader& Shader);` (private)
- **Purpose:** Configure OpenGL state (textures, lighting, vertex attributes) for a single render pass.
- **Inputs:** Model reference, shader configuration reference
- **Outputs/Return:** void
- **Side effects:** Calls texture and lighting callbacks; modifies OpenGL state
- **Notes:** Implementation not visible in header

### Clear
- **Signature:** `void Clear();`
- **Purpose:** Reset persistent arrays for a fresh rendering cycle.
- **Outputs/Return:** void
- **Side effects:** Clears IndexedCentroidDepths, SortedVertIndices, ExtLightColors vectors

## Notes on Public Members
- **ViewDirection[3]** (GLfloat): External-facing array for camera view direction in model coordinates; used to compute polygon centroid depths for sorting
- **Render flags enum:** Textured, Colored, ExtLight (enable external-light colors), EL_SemiTpt (semitransparency in external lights)

## Control Flow Notes
Part of the render phase. Invoked after model geometry and frame/sequence state are set up in Model3D. Polygon depth-sorting only applies when Z-buffer is absent; otherwise, separable shaders may render independently. Non-separable and semitransparent shaders always depth-sort.

## External Dependencies
- `Model3D.h`: Defines 3D model structure (vertices, bones, frames, sequences, normals, texture coordinates, colors)
- **OpenGL types:** GLfloat, GLushort, GLfloat arrays (vertices, normals, colors, texture coordinates)
- **STL:** `vector<>` for dynamic arrays
