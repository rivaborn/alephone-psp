# Source_Files/Network/network_capabilities.cpp

## File Purpose
Defines static string constants for the Capabilities versioning system, used as keys to store and query network protocol and feature support versions during multiplayer game sessions.

## Core Responsibilities
- Initialize static string constant keys for capability lookup in the Capabilities map
- Provide named constants for protocol and feature versioning (Gameworld, Star, Ring, Lua, Speex, Gatherable, ZippedData)

## Key Types / Data Structures
None (Capabilities class defined in header only).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Capabilities::kGameworld` | `const string` | static | Key for PRNG, physics, core simulation versioning |
| `Capabilities::kStar` | `const string` | static | Key for Star network protocol version |
| `Capabilities::kRing` | `const string` | static | Key for Ring network protocol version |
| `Capabilities::kLua` | `const string` | static | Key for Lua scripting support version |
| `Capabilities::kSpeex` | `const string` | static | Key for Speex network microphone version |
| `Capabilities::kGatherable` | `const string` | static | Key for joiner's gatherable capability version |
| `Capabilities::kZippedData` | `const string` | static | Key for zipped data (map/lua/physics) support version |

## Key Functions / Methods
None.

## Control Flow Notes
Pure initialization fileΓÇöconstants are initialized at program startup. No runtime control flow.

## External Dependencies
- `#include "network_capabilities.h"` ΓÇö defines Capabilities class and version constants
- `cseries.h` (via header) ΓÇö standard definitions
- `<string>`, `<map>` (standard library) ΓÇö used in Capabilities typedef
