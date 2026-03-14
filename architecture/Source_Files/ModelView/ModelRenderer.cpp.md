# Source_Files/ModelView/ModelRenderer.cpp

## File Purpose
Implements 3D model rendering with support for multipass shading, depth-sorted polygon rendering (when Z-buffer unavailable), and shader callbacks for texturing and external lighting. Part of the Aleph One game engine.

## Core Responsibilities
- Render 3D models with multiple shader passes and optimization based on Z-buffer availability
- Compute centroid depths and depth-sort triangles for proper back-to-front rendering without Z-buffer
- Configure OpenGL state for texturing, color arrays, and external lighting per render pass
- Manage internal scratch buffers (centroid indices, sorted vertex indices, lighting colors) to avoid repeated allocation
- Delegate texture and lighting computation to external callbacks for shader flexibility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ModelRenderer | class | Main renderer; owns render pipeline and scratch buffers |
| ModelRenderShader | struct (defined in .h) | Shader descriptor with flags and texture/lighting callbacks |
| IndexedCentroidDepth | struct (defined in .h) | Triangle index + depth for sorting; operator< sorts far-to-near |
| Model3D | class (defined elsewhere) | 3D mesh with positions, normals, colors, texture coords, indices |

## Global / File-Static State
None.

## Key Functions / Methods

### Render
- **Signature:** `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer)`
- **Purpose:** Main entry point; render a 3D model with one or more shader passes, optimizing based on Z-buffer and shader separability.
- **Inputs:**
  - `Model`: 3D mesh (positions, normals, colors, texture coords, vertex indices)
  - `Shaders`: array of shader configurations
  - `NumShaders`: count of shaders to apply
  - `NumSeparableShaders`: number of shaders (from start of array) that can be rendered in one pass when Z-buffer present; nonseparable shaders require per-triangle rendering
  - `Use_Z_Buffer`: if true, Z-buffer enables separability optimization; if false, forces depth-sorting regardless
- **Outputs/Return:** None; produces OpenGL draw calls.
- **Side effects:**
  - Populates `IndexedCentroidDepths` and `SortedVertIndices` member vectors
  - Modifies OpenGL state: enables vertex array, texture/color arrays, issues draw calls
  - Modifies `NumSeparableShaders` parameter (called by value, so local copy)
- **Calls:**
  - OpenGL: `glEnableClientState`, `glVertexPointer`, `glDisableClientState`, `glDrawElements`, `glDisable`, `glEnable`
  - STL: `std::sort` on centroid-depth array
  - `SetupRenderPass` (internal)
- **Notes:**
  - Early returns if no shaders, null shader array, or empty model.
  - Three rendering strategies:
    1. **All separable** (Z-buffer present): one pass per shader, all triangles in one draw call per pass.
    2. **Mixed** (Z-buffer, some nonseparable): separable shaders depth-sorted in one pass each; nonseparable shaders rendered per-triangle after depth sort.
    3. **All nonseparable** (no Z-buffer): all triangles depth-sorted; each triangle rendered with each shader.
  - Centroid depth: dot product of triangle centroid (averaged vertex positions) with `ViewDirection` array (must be set by caller).
  - Optimization: if exactly one shader is nonseparable, it is treated as separable (no per-triangle loop).

### SetupRenderPass
- **Signature:** `void SetupRenderPass(Model3D& Model, ModelRenderShader& Shader)`
- **Purpose:** Configure OpenGL state (textures, colors, lighting) for a single rendering pass.
- **Inputs:**
  - `Model`: 3D mesh
  - `Shader`: shader configuration with flags and callbacks
- **Outputs/Return:** None; modifies OpenGL state.
- **Side effects:**
  - Enables/disables `GL_TEXTURE_2D` and texture coordinate arrays based on model and shader flags.
  - If external lighting requested and model has normals, invokes `Shader.LightingCallback` to populate `ExtLightColors`.
  - If model has colors and shader requests colored rendering, multiplies external light colors by model colors (per-component for RGB, skips alpha in RGBA).
  - Invokes `Shader.TextureCallback` for custom texture setup.
  - Populates/resizes `ExtLightColors` scratch buffer.
- **Calls:**
  - OpenGL: `glEnable`, `glDisable`, `glEnableClientState`, `glDisableClientState`, `glTexCoordPointer`, `glColorPointer`
  - `Shader.LightingCallback(data, NumVerts, normals, positions, colors)` (external, user-provided)
  - `Shader.TextureCallback(data)` (external, user-provided)
- **Notes:**
  - Asserts that `TextureCallback` is non-null.
  - Lighting callback receives normals and positions in model coordinates.
  - Color multiplication: 3-component loop multiplies all values; 4-component loop skips alpha channel (index 3 of each 4-tuple).

### Clear
- **Signature:** `void Clear()`
- **Purpose:** Deallocate internal scratch buffers.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` vectors.
- **Calls:** `vector::clear()` on three member vectors.
- **Notes:** Called to reset state or release memory between rendering sessions.

## Control Flow Notes
- **Entry point:** `Render()` called once per frame per model.
- **Z-buffer decision:** Caller decides whether Z-buffer is available; if not, renderer must depth-sort all polygons.
- **Separability:** Separable shaders (typically opaque passes) can be rendered in a single pass if Z-buffer available. Nonseparable shaders (typically semitransparent) require per-triangle rendering to ensure correct blending order.
- **Depth metric:** Uses viewer-relative depth (dot product with `ViewDirection`, set by caller) for back-to-front sort.
- **Rendering loop:** Setup pass ΓåÆ draw triangles (separable), or setup per-triangle ΓåÆ draw 3 indices (nonseparable).

## External Dependencies
- **OpenGL** (conditional on `HAVE_OPENGL` define): vertex arrays, draw elements, state flags.
- **STL:** `<algorithm>` for `std::sort`, `<vector>` for dynamic arrays.
- **Local headers:** `"ModelRenderer.h"` (class definition), `"cseries.h"` (Aleph One platform abstractions).
- **Defined elsewhere:** `Model3D` class, `ModelRenderShader` struct, `TEST_FLAG` macro, `obj_clear()` macro.
