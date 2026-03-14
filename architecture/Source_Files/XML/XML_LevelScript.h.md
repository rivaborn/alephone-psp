# Source_Files/XML/XML_LevelScript.h

## File Purpose
Provides interfaces for loading and executing XML-based level scripts in the Aleph One engine. Supports level-specific MML (Markup Macro Language) execution, movie file management, and end-game sequence handling. Designed to execute scripting commands on a per-level basis during game initialization and level progression.

## Core Responsibilities
- Load level scripts from map file resources
- Execute level-specific MML scripts and Pfhortran interpreter setup
- Manage end-of-game sequences and restoration scripts
- Locate and cache movie files for level playback and end-game cinematics
- Provide script parser interface for external XML element parsing
- Configure end-game screens (selection and count)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (from FileHandler.h) | File path abstraction; used to specify map files and movies |
| `XML_ElementParser` | class (forward declared) | Parser for XML script elements; implementation defined elsewhere |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EndScreenIndex` | short | global extern | Fake level index for end-of-game screens |
| `NumEndScreens` | short | global extern | Number of end-of-game screens to display in sequence |

## Key Functions / Methods

### LoadLevelScripts
- **Signature:** `void LoadLevelScripts(FileSpecifier& MapFile);`
- **Purpose:** Load all level scripts (from resource 128) in a map file into memory.
- **Inputs:** MapFile ΓÇö reference to file specifier pointing to the map
- **Outputs/Return:** void
- **Side effects:** Loads scripts into memory; populates internal level-script database
- **Calls:** Not inferable from this file
- **Notes:** Called during map initialization; uses resource 128 or platform-appropriate equivalent

### RunLevelScript
- **Signature:** `void RunLevelScript(int LevelIndex);`
- **Purpose:** Execute level-specific scripts, including Pfhortran interpreter setup and MML execution.
- **Inputs:** LevelIndex ΓÇö integer index identifying the level
- **Outputs/Return:** void
- **Side effects:** Executes scripts; modifies game state (physics, AI, visuals, etc.)
- **Calls:** Not inferable from this file
- **Notes:** Typically called at level start; applies all level-specific customizations

### RunEndScript
- **Signature:** `void RunEndScript();`
- **Purpose:** Execute script designated for game conclusion/end state.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies game state for end sequence
- **Calls:** Not inferable from this file

### RunRestorationScript
- **Signature:** `void RunRestorationScript();`
- **Purpose:** Restore default parameter values modified by MML across various game systems.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Resets parameters to engine defaults
- **Calls:** Not inferable from this file
- **Notes:** Provides cleanup mechanism for MML-induced parameter changes; intended as convenience utility

### FindLevelMovie
- **Signature:** `void FindLevelMovie(short index);`
- **Purpose:** Locate and cache movie files for a specific level and end-of-game playback.
- **Inputs:** index ΓÇö level index (short)
- **Outputs/Return:** void (caches result internally)
- **Side effects:** Searches file system and caches movie file specifiers
- **Calls:** Not inferable from this file
- **Notes:** Separates level movie from end-game movie; supports distinct playback sequences

### GetLevelMovie
- **Signature:** `FileSpecifier *GetLevelMovie(float& PlaybackSize);`
- **Purpose:** Retrieve cached movie file specifier for current level.
- **Inputs:** PlaybackSize ΓÇö reference to float; playback scale (output parameter, not modified if unspecified)
- **Outputs/Return:** FileSpecifier pointer; NULL if no movie available
- **Side effects:** May populate PlaybackSize with stored scale value
- **Calls:** Not inferable from this file
- **Notes:** Returns NULL on absence; PlaybackSize modified only if explicitly set in script

### ExternalDefaultLevelScript_GetParser
- **Signature:** `XML_ElementParser *ExternalDefaultLevelScript_GetParser();`
- **Purpose:** Retrieve XML element parser for default external level scripts.
- **Inputs:** None
- **Outputs/Return:** Pointer to XML_ElementParser
- **Calls:** Not inferable from this file
- **Notes:** Factory function; parser instance lifetime/ownership not specified

## Control Flow Notes
Integrates into game level-loading and shutdown pipelines:
- **Level Start:** `LoadLevelScripts` (once per map) ΓåÆ `RunLevelScript` (per level) ΓåÆ `FindLevelMovie`
- **Game End:** `RunEndScript`
- **System Cleanup:** `RunRestorationScript` resets engine state post-level

## External Dependencies
- **FileHandler.h** ΓÇö provides `FileSpecifier` class for file/path abstraction
- **XML parser** ΓÇö `XML_ElementParser` (forward declared; implementation defined elsewhere)
- **Pfhortran scripting engine** ΓÇö referenced in comments; integration points not visible in this header
- **MML system** ΓÇö supports Markup Macro Language execution; no declarations in this header
