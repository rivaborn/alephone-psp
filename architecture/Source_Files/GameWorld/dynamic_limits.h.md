# Source_Files/GameWorld/dynamic_limits.h

## File Purpose
Defines the interface for managing dynamically-configured resource limits in the game engine. This header exposes functions to query limits on game entities (objects, NPCs, projectiles, effects, etc.) whose values are initialized from XML configuration files rather than compile-time constants.

## Core Responsibilities
- Define enumeration of all limit types (objects, monsters, projectiles, effects, collision buffers, etc.)
- Provide XML parser integration for loading limit configurations
- Expose accessor interface to retrieve current limits at runtime

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_dynamic_limit_*` | enum constants | Identifiers for 8 distinct resource limit types (e.g., `_dynamic_limit_objects`, `_dynamic_limit_monsters`, `_dynamic_limit_projectiles`, `_dynamic_limit_effects`, `_dynamic_limit_rendered`, `_dynamic_limit_local_collision`, `_dynamic_limit_global_collision`) |
| `NUMBER_OF_DYNAMIC_LIMITS` | enum constant | Sentinel value; count of limit types |

## Global / File-Static State
None.

## Key Functions / Methods

### DynamicLimits_GetParser
- Signature: `XML_ElementParser *DynamicLimits_GetParser()`
- Purpose: Return the XML parser object that handles parsing and loading dynamic limit configuration from XML
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser for dynamic limits
- Side effects: Implementation likely initializes or returns a singleton parser instance
- Calls: Not visible (implementation in .cpp)
- Notes: Called during engine initialization to set up XML configuration parsing

### get_dynamic_limit
- Signature: `uint16 get_dynamic_limit(int which)`
- Purpose: Query the current value of a specific resource limit
- Inputs: `which` ΓÇô index matching one of the `_dynamic_limit_*` enum constants
- Outputs/Return: uint16 ΓÇô the limit value for the requested resource type
- Side effects: None (pure accessor)
- Calls: Not visible (implementation in .cpp)
- Notes: Called at runtime to determine how many of a resource type can be active; invalid `which` behavior not defined in header

## Control Flow Notes
This file is part of initialization and resource-management subsystems. The engine likely:
1. Calls `DynamicLimits_GetParser()` during startup to prepare XML parsing
2. Loads XML configuration files containing limit values
3. Queries `get_dynamic_limit()` at runtime (e.g., object spawning, effect allocation) to enforce caps

No per-frame or render-time control flow visible.

## External Dependencies
- `XML_ElementParser.h` ΓÇô base parser class for XML configuration system
- Standard C headers (implicit: `<stdio.h>` in bundled XML_ElementParser.h)
- Implementation file (not provided) likely includes engine-specific types and storage
