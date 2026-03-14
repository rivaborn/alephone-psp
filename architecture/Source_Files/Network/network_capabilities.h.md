# Source_Files/Network/network_capabilities.h

## File Purpose

Defines a versioning system for network capabilities and game engine features in Aleph One. The `Capabilities` class wraps a map of feature names to version numbers, enabling peers to negotiate protocol compatibility during network setup (gathering, joining, or protocol exchanges).

## Core Responsibilities

- Define version constants for core network protocols (Gameworld PRNG/physics, Star, Ring)
- Define version constants for optional engine features (Lua scripting, Speex audio, zipped data support)
- Provide a type-safe key-value store (`Capabilities` class) for capability versioning
- Enforce a maximum key size limit (1024 bytes) to prevent buffer issues
- Enable capability negotiation by exporting standardized capability names

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `capabilities_t` | typedef | Alias for `map<string, uint32>` storing capability names ΓåÆ version numbers |
| `Capabilities` | class | Public subclass of `capabilities_t` with bounds-checked `operator[]` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kGameworldVersion` | static const int | class-level | Version for PRNG and physics engine (value: 1) |
| `kStarVersion` | static const int | class-level | Version for Star network protocol (value: 5) |
| `kRingVersion` | static const int | class-level | Version for Ring network protocol (value: 1) |
| `kLuaVersion` | static const int | class-level | Version for Lua scripting support (value: 2) |
| `kSpeexVersion` | static const int | class-level | Version for Speex network microphone (value: 1) |
| `kGatherableVersion` | static const int | class-level | Version for joiner "gatherable" status (value: 1) |
| `kZippedDataVersion` | static const int | class-level | Version for compressed map/lua/physics data (value: 1) |
| `kGameworld` | static const string | class-level | Key name for gameworld capability |
| `kStar` | static const string | class-level | Key name for Star protocol capability |
| `kRing` | static const string | class-level | Key name for Ring protocol capability |
| `kLua` | static const string | class-level | Key name for Lua capability |
| `kSpeex` | static const string | class-level | Key name for Speex audio capability |
| `kGatherable` | static const string | class-level | Key name for joiner gatherable status |
| `kZippedData` | static const string | class-level | Key name for zipped data capability |

## Key Functions / Methods

### operator[]
- **Signature:** `uint32& operator[](const string& k)`
- **Purpose:** Overrides the inherited `map::operator[]` to add bounds checking on key length
- **Inputs:** Key name (`k`) as `const string&`
- **Outputs/Return:** Reference to `uint32` value (version number)
- **Side effects (global state, I/O, alloc):** Potential map insertion if key doesn't exist (inherited from `std::map`)
- **Calls (direct calls visible in this file):** `capabilities_t::operator[](k)` (parent class method)
- **Notes:** Asserts that key length is under `kMaxKeySize` (1024); prevents oversized keys during capability negotiation

## Control Flow Notes

This is a pure data structure header with no control flow. It is used during network initialization/negotiation phases when peers exchange capability versions to determine protocol compatibility. Version numbers allow protocol evolution while maintaining backward/forward compatibility checks.

## External Dependencies

- **cseries.h**: Provides platform/compiler abstractions and common types (e.g., `uint32`)
- **\<string\>**: Standard C++ string class
- **\<map\>**: Standard C++ map container
