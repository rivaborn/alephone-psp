# Subsystem Overview

## Purpose
The ModelView subsystem manages loading, storage, and rendering of 3D skeletal animated models in the Aleph One/Marathon engine. It supports multiple model file formats (Dim3 XML, Wavefront OBJ, 3D Studio Max 3DS, QuickDraw 3D) and provides OpenGL-based rendering with animation frame interpolation and multipass shader support.

## Key Files

| File | Role |
|------|------|
| Model3D.h | Core data structures for 3D geometry, skeletal bones, animation frames, and sequences |
| Model3D.cpp | Skeletal deformation, frame interpolation, normal computation, and bounding box management |
| ModelRenderer.h | Rendering interface with multipass shader support and depth-sorted polygon rendering |
| ModelRenderer.cpp | OpenGL rendering implementation with Z-buffer and depth-sort fallback modes |
| Dim3_Loader.h / Dim3_Loader.cpp | Dim3 XML format parser for loading models with bone hierarchies and animation |
| WavefrontLoader.h / WavefrontLoader.cpp | Wavefront OBJ format parser with vertex deduplication and polygon triangulation |
| StudioLoader.h / StudioLoader.cpp | 3D Studio Max 3DS binary format parser with chunk-based hierarchical parsing |
| QD3D_Loader.h | QuickDraw 3D/Quesa format loader with tesselation configuration |

## Core Responsibilities

- Load 3D models from multiple file formats (Dim3, OBJ, 3DS, QD3D) into unified Model3D structures
- Store and manage skeletal bone hierarchies with parent-child relationships and depth-first traversal ordering
- Organize animation frames and sequences with vertex source mapping for bone deformation
- Apply skeletal deformations to vertices using bone transformations and frame interpolation with blending
- Compute vertex and per-polygon normals with smoothing and vertex splitting options
- Render 3D models via multipass OpenGL shaders with texture and lighting callbacks
- Perform depth-sorted polygon rendering by centroid when Z-buffer is unavailable
- Maintain and compute bounding boxes for models and provide debug visualization support
- Handle cross-platform file I/O through FileSpecifier abstraction

## Key Interfaces & Data Flow

**Exposes:**
- `Model3D` structure containing geometry (vertex positions, normals, texture coordinates, colors) and skeletal animation data
- `ModelRenderer` class for rendering Model3D objects with configurable shader callbacks
- Loader functions (`Load_Dim3_Model`, `Load_Studio_Model`, `Load_Wavefront_Model`, etc.) that consume FileSpecifier and populate Model3D
- Debug output configuration interfaces for all loaders

**Consumes:**
- File I/O via `FileSpecifier` and `OpenedFile` abstractions from FileHandler subsystem
- OpenGL rendering types and function calls (conditional on `HAVE_OPENGL`)
- Precomputed trigonometric tables and angle constants from world.h
- Vector math operations (VecCopy, VecAdd, VecSub, scalar products) from VecOps.h
- Platform abstractions and utility macros from cseries.h
- XML parsing via Expat library (for Dim3 format)
- Binary stream unpacking via Packing.h endian-aware macros

## Runtime Role

Models are typically loaded during level initialization or resource loading phases via the format-specific loader functions. During each frame, `ModelRenderer` transforms model geometry according to current animation frame state (via `Model3D` interpolation), applies shader callbacks for texturing and lighting, and submits geometry to OpenGL. The renderer selects between Z-buffered or depth-sorted rendering modes based on Z-buffer availability. Bone hierarchies are reordered once at load time for efficient depth-first traversal during animation application.

## Notable Implementation Details

- **Multipass rendering:** ModelRenderer supports separable and non-separable shader passes with per-pass OpenGL state configuration to avoid redundant transformations
- **Depth-sorting fallback:** When Z-buffer is unavailable, triangles are sorted by centroid depth and rendered back-to-front; centroid and sorted index arrays are cached across frames to reduce allocation overhead
- **Vertex deduplication:** WavefrontLoader identifies unique position/texcoord/normal combinations to merge redundant vertices and reduce GPU load
- **Bone reordering:** Dim3_Loader reorders bones into depth-first traversal order for cache-efficient animation application
- **Frame interpolation:** Model3D supports smooth blending between animation frames with configurable crossfading
- **Format flexibility:** Support for five distinct model formats allows artists to use their preferred 3D tools (3DS Max, Blender via OBJ, Quesa, Dim3)
