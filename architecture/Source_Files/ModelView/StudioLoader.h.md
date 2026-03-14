# Source_Files/ModelView/StudioLoader.h

## File Purpose
Header file declaring the interface for loading Autodesk 3D Studio Max (.3ds) model files into the engine's Model3D representation. Provides functions to load 3D models from disk and configure debug message output during loading.

## Core Responsibilities
- Declare the main entry point for 3DS model file loading
- Provide a global debug output configuration mechanism
- Abstract 3DS format parsing and file I/O from callers

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model);`
- **Purpose:** Load a 3D Studio Max model file and populate a Model3D structure with vertex, normal, bone, frame, and animation data.
- **Inputs:** FileSpecifier reference (file path/handle); Model3D reference (output container)
- **Outputs/Return:** Boolean success/failure status
- **Side effects:** Modifies Model3D in-place; reads from disk; may emit debug messages via SetDebugOutput_Studio() destination
- **Calls:** Not inferable from this file (implementation in .cpp)
- **Notes:** Primary loader entry point; implementation likely parses 3DS binary format and populates Model3D's position, normal, texture coordinate, bone, frame, and sequence arrays

### SetDebugOutput_Studio
- **Signature:** `void SetDebugOutput_Studio(FILE *DebugOutput);`
- **Purpose:** Configure the global destination for status/debug messages emitted during 3DS loading.
- **Inputs:** FILE pointer (null to disable; typically stdout/stderr or a log file handle)
- **Outputs/Return:** void
- **Side effects:** Sets global debug output stream (not thread-safe)
- **Calls:** Not inferable from this file
- **Notes:** Allows callers to redirect loader diagnostics; optional (loader likely works without calling this)

## Control Flow Notes
Typical usage is during engine initialization/model loading phase:
1. Optionally call SetDebugOutput_Studio() once to enable logging
2. Call LoadModel_Studio() for each 3DS file needed
3. Use the populated Model3D in rendering/animation subsystems

No explicit shutdown is required for this module.

## External Dependencies
- `<stdio.h>` ΓÇö for FILE type  
- `Model3D.h` ΓÇö defines Model3D struct containing positions, normals, texture coordinates, bones, frames, sequences, and bounding box  
- `FileHandler.h` ΓÇö defines FileSpecifier for cross-platform file abstraction  
- Implementation details (3DS binary format parsing) in StudioLoader.cpp (not provided)
