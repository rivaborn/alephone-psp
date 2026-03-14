# Source_Files/XML/XML_MakeRoot.cpp

## File Purpose
Constructs and initializes the root of the XML parser tree for the Marathon game engine. It defines the global root and "marathon" element parsers, registers all subsystem XML parsers as children, and provides functions to set up the complete parse tree and reset MML-configured values to defaults.

## Core Responsibilities
- Define the absolute XML document root element
- Create the canonical "marathon" element as the top-level game configuration root
- Register ~30 subsystem parsers as children of the marathon element
- Build the complete hierarchical XML parse tree during engine initialization
- Provide a reset mechanism to restore all MML-configured values to hard-coded defaults

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ElementParser` | class | Element parser node; defined elsewhere; used for hierarchical XML tree structure |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RootParser` | `XML_ElementParser` | global | Absolute document root element (contains all valid XML document roots) |
| `MarathonParser` | `XML_ElementParser` | global | Canonical root element for Marathon XML configuration files |

## Key Functions / Methods

### SetupParseTree
- **Signature:** `void SetupParseTree()`
- **Purpose:** Register all subsystem XML parsers as children of the marathon element, establishing the complete parse tree hierarchy.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Modifies global `RootParser` and `MarathonParser` by adding ~30 child parsers; must be called before any XML parsing occurs
- **Calls:** 
  - `RootParser.AddChild(&MarathonParser)`
  - `MarathonParser.AddChild(...)` ├ù ~30 (text strings, interface, player, items, weapons, sounds, rendering, input, etc.)
- **Notes:** Entry point for XML infrastructure initialization. Registers parsers for: TS (text strings), Interface, PlayerName, Infravision, MotionSensor, OverheadMap, DynamicLimits, AnimatedTextures, Player, Items, ControlPanels, Liquids, Sounds, Platforms, Scenery, Faders, View, Landscapes, Weapons, OpenGL, Cheats, TextureLoading, Keyboard, DamageKicks, Logging, Scenario, Theme (SDL only), SW_Texture_Extras, Console, and ExternalDefaultLevelScript.

### ResetAllMMLValues
- **Signature:** `void ResetAllMMLValues()`
- **Purpose:** Reset all MML (Marathon Markup Language) configuration values across all subsystems to hard-coded defaults.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Traverses and resets state in all registered child parsers via the parse tree
- **Calls:** `MarathonParser.ResetChildrenValues()`
- **Notes:** Operates on entire registered tree; used when reverting to default configuration state.

## Control Flow Notes
Initialization phase: `SetupParseTree()` must be called during engine startup before parsing any XML configuration files. The resulting parse tree structure enables hierarchical XML parsing of game configuration. `ResetAllMMLValues()` is called to restore clean state, likely when reverting game settings or restarting.

## External Dependencies
- **XML infrastructure:** `XML_ElementParser`, `XML_ParseTreeRoot.h` (root declarations; class definition elsewhere)
- **Subsystem parsers (all `*_GetParser()` functions defined in respective modules):**
  - Game logic: `Player`, `Items`, `Weapons`, `Monsters`, `Scenery`, `Platforms`, `Liquids`
  - Rendering/UI: `Interface`, `MotionSensor`, `OverheadMap`, `AnimatedTextures`, `Landscapes`, `OpenGL`, `SW_Texture_Extras`
  - Input/Output: `Keyboard`, `Console`, `SoundManager`, `Logging`
  - Configuration: `TextStrings`, `PlayerName`, `DynamicLimits`, `Scenario`, `ExternalDefaultLevelScript`
  - Conditionals: `Theme_GetParser()` (SDL only)
- **Engine headers:** `world.h`, `interface.h`, `game_window.h`, `vbl.h`, `shell.h` (platform/window management)
