# Source_Files/Files/filetypes_macintosh.cpp

## File Purpose
Manages macOS file typecodes for the Aleph One game engine, mapping between OSType (macOS four-character file type codes) and internal Typecode enums. Loads custom typecodes from the resource fork and maintains backwards compatibility with Marathon 2 file types.

## Core Responsibilities
- Load custom FTyp resource from macOS resource fork (resource 128)
- Maintain bidirectional mapping between OSType and Typecode
- Support multiple OSType mappings to single internal Typecode (M2 compatibility)
- Provide accessors to query and modify typecode mappings
- Initialize runtime lookup map on startup
- Handle boundary conditions and unsupported typecodes gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `file_type_to_a1_typecode_rec` | struct | Maps a single OSType to a Typecode |
| `file_type_to_a1_typecode_t` | typedef | Runtime map (std::map) enabling O(log n) OSTypeΓåÆTypecode lookup |
| `Typecode` | enum | Internal filetype identifier (defined in tags.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `typecodes[]` | OSType[NUMBER_OF_TYPECODES] | static | Primary typecode array indexed by Typecode enum |
| `additional_typecodes[]` | file_type_to_a1_typecode_rec[] | static | M2-compatible + A1-specific OSType mappings |
| `file_type_to_a1_typecode` | map<OSType, Typecode> | static | Runtime lookup map; built during initialization |

## Key Functions / Methods

### initialize_typecodes()
- **Signature:** `void initialize_typecodes()`
- **Purpose:** Initialize the typecode system by loading custom codes from resource fork and building the runtime lookup map.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies global `typecodes[]` array and `file_type_to_a1_typecode` map; accesses macOS resource fork via GetResource/HLock/HUnlock/ReleaseResource.
- **Calls:** GetResource, GetHandleSize, HLock, HUnlock, ReleaseResource (macOS API)
- **Notes:** Skipped on SDL builds without SDL_RFORK_HACK. Performs memcpy from resource data; bounds-checks FTyp size against sizeof(typecodes).

### get_typecode()
- **Signature:** `OSType get_typecode(Typecode which)`
- **Purpose:** Retrieve the OSType for a given internal Typecode.
- **Inputs:** Typecode index
- **Outputs/Return:** OSType (4-character code); returns '????' for out-of-range indices
- **Side effects:** None
- **Calls:** None
- **Notes:** Bounds-checked; safe for invalid enum values.

### set_typecode()
- **Signature:** `void set_typecode(Typecode which, OSType _type)`
- **Purpose:** Set or override the OSType for a given Typecode.
- **Inputs:** Typecode index, new OSType value
- **Outputs/Return:** None
- **Side effects:** Modifies global `typecodes[]` array
- **Calls:** None
- **Notes:** Silently ignores out-of-bounds writes (no error raised).

### get_typecode_for_file_type()
- **Signature:** `Typecode get_typecode_for_file_type(OSType inType)`
- **Purpose:** Reverse lookup: given an OSType, return the corresponding internal Typecode.
- **Inputs:** OSType (macOS file type code)
- **Outputs/Return:** Typecode; returns `_typecode_unknown` if not found
- **Side effects:** None (read-only lookup)
- **Calls:** map::find() via iterator
- **Notes:** Asserts that `file_type_to_a1_typecode` is non-empty (requires initialize_typecodes() to be called first). O(log n) lookup.

### get_all_file_types_for_typecode()
- **Signature:** `const vector<OSType> get_all_file_types_for_typecode(Typecode which)`
- **Purpose:** Retrieve all OSType values that map to a given Typecode (handles 1ΓåÆmany mapping).
- **Inputs:** Typecode
- **Outputs/Return:** vector<OSType> (possibly empty)
- **Side effects:** Allocates and returns a new vector
- **Calls:** vector::push_back()
- **Notes:** Checks primary `typecodes[]` first, then scans `additional_typecodes[]`. Filters out '????' pseudo-typecodes.

## Control Flow Notes
This file is conditionally compiled (`#if defined(mac) || ...SDL_RFORK_HACK`) and is part of macOS startup initialization. `initialize_typecodes()` must be called during engine startup (before file I/O). The global `file_type_to_a1_typecode` map is populated once at initialization and used for file-type discovery throughout the application lifetime (e.g., in environment/preferences dialogs).

## External Dependencies
- **macOS API:** GetResource, GetHandleSize, HLock, HUnlock, ReleaseResource (Carbon/resource management)
- **Standard library:** std::map, std::vector, string.h
- **Engine headers:** tags.h (Typecode enum definition), csalerts.h (assert/alert macros)
- **Conditional:** Carbon/Carbon.h (if EXPLICIT_CARBON_HEADER defined)
