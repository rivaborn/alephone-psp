# Source_Files/RenderOther/overhead_map.h

## File Purpose
Defines the interface for rendering overhead maps in multiple modes (saved game preview, checkpoint, game map). Declares the data structure for overhead map configuration and provides access to XML configuration parsing for this subsystem.

## Core Responsibilities
- Define overhead map rendering modes and scale constraints
- Declare the configuration data structure for overhead map rendering
- Provide the main rendering entry point
- Expose XML parser for loading overhead map settings from configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `overhead_map_data` | struct | Aggregates all configuration for rendering an overhead map: scale, origin, dimensions, clipping bounds, and render mode |

## Global / File-Static State
None.

## Key Functions / Methods

### _render_overhead_map
- Signature: `void _render_overhead_map(struct overhead_map_data *data)`
- Purpose: Render the overhead map using the provided configuration
- Inputs: Pointer to `overhead_map_data` struct containing scale, origin, dimensions, and render mode
- Outputs/Return: None (void)
- Side effects: Renders to framebuffer; modifies rendering state
- Calls: Not inferable from this file
- Notes: Implementation location unknown; likely in a paired `.c` file

### OverheadMap_GetParser
- Signature: `XML_ElementParser *OverheadMap_GetParser()`
- Purpose: Retrieve the XML element parser for "overhead_map" configuration elements
- Inputs: None
- Outputs/Return: Pointer to `XML_ElementParser` object configured for overhead map elements
- Side effects: None (getter)
- Calls: Not inferable from this file
- Notes: Added by Loren Petrich for XML configuration support; implementation location unknown

## Control Flow Notes
This file defines the interface layer. Configuration is loaded via XML parsing (using `OverheadMap_GetParser()`); rendering occurs via `_render_overhead_map()` call, likely during the render phase of the main game loop. The three modes suggest different rendering contexts (preview vs. gameplay).

## External Dependencies
- `XML_ElementParser.h` ΓÇô base class for XML element parsing; provides hierarchical configuration parsing infrastructure
- No standard library includes visible; likely uses game engine type definitions (`world_point2d`, `short`) from elsewhere
