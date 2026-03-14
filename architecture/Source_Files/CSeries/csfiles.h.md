# Source_Files/CSeries/csfiles.h

## File Purpose
Header file declaring file specification and filesystem utility functions for the CSeries compatibility layer. Provides abstraction for classic macOS file operations (FSSpec-based) used throughout the engine.

## Core Responsibilities
- Declare file specification retrieval from resource lists
- Declare application-local file specification acquisition
- Provide macOS filesystem abstraction interface

## Key Types / Data Structures
None (declarations only; FSSpec and OSErr are defined elsewhere).

## Global / File-Static State
None.

## Key Functions / Methods

### get_file_spec
- Signature: `OSErr get_file_spec(FSSpec *spec, short listid, short item, short pathsid)`
- Purpose: Retrieve a file specification from a resource list or path list
- Inputs: resource list ID, item index within list, path resource ID
- Outputs/Return: OSErr status code; populates FSSpec pointer with resolved file spec
- Side effects: Reads resources; modifies caller's FSSpec structure
- Calls: Not inferable from this file
- Notes: Returns error code; caller responsible for FSSpec pointer validity

### get_my_fsspec
- Signature: `OSErr get_my_fsspec(FSSpec *spec)`
- Purpose: Retrieve the current application's own file specification
- Inputs: Output pointer for FSSpec
- Outputs/Return: OSErr status code; populates FSSpec with application's own file
- Side effects: Modifies caller's FSSpec structure
- Calls: Not inferable from this file
- Notes: Likely used for accessing bundled resources or locating game data relative to executable

## Control Flow Notes
Utility header; no control flow. Functions are called from throughout the engine for file operations and resource location.

## External Dependencies
- `OSErr` ΓÇö macOS classic error type (defined elsewhere)
- `FSSpec` ΓÇö classic macOS file specification structure (defined elsewhere)
- License: GNU GPL v2+; copyright Bo Lindbergh and Aleph One contributors
