# Subsystem Overview

## Purpose

The XML subsystem orchestrates parsing of game configuration files (MML-derived markup) using the Expat C parser library. It implements a hierarchical tree of element-specific parsers that convert XML attributes into game engine data structures (colors, damage definitions, shapes, level scripts, music, and UI configurations) during engine initialization and level loading.

## Key Files

| File | Role |
|------|------|
| XML_Configure.cpp/h | Expat parser lifecycle management; element parser tree navigation; callback routing from C API to C++ |
| XML_ElementParser.cpp/h | Base class providing typed value parsing (int/uint/float/bool), attribute validation, element hierarchy management, UTF-8 conversion |
| XML_DataBlock.cpp/h | Memory buffer parser; error reporting (read/parse/interpretation) with platform-specific dialogs and logging |
| XML_Loader_SDL.cpp/h | Disk file loading and directory scanning; integrates FileSpecifier I/O; reports errors with filename/line context |
| ColorParser.cpp/h | Parses `<color>` elements; converts float [0ΓÇô1] channels to uint16; indexed array storage |
| DamageParser.cpp/h | Parses `<damage>` elements; type/flag validation; floating-point to fixed-point conversion |
| ShapesParser.cpp/h | Parses `<shape>` elements; validates collection/CLUT/sequence/frame; composes shape_descriptor |
| XML_LevelScript.cpp/h | Loads level scripts from map resources; executes MML, Lua, music, movies, load screens per-level and globally (Default/Restore/End) |
| XML_MakeRoot.cpp | Constructs root element parser; registers ~30 subsystem parsers as children |
| XML_ParseTreeRoot.h | Declares global RootParser; exports SetupParseTree(), ResetAllMMLValues() |

## Core Responsibilities

- Manage Expat parser creation, lifecycle, and attribute/text callbacks via static C wrappers
- Maintain a stack of active element parsers during XML traversal; route semantic interpretation to subsystem-specific parsers
- Parse and validate typed XML attribute values (integers with bounds checking, floats, booleans, strings)
- Convert float color channels and fixed-point scale values to engine-native formats
- Load XML configuration files from disk (FileSpecifier abstraction) and filter backup/script files
- Report three-tier error hierarchy: read errors (fatal, platform dialogs), XML parse errors (Expat, with line numbers), interpretation errors (semantic validation, with logging throttle)
- Initialize parser tree and reset MML-configured values to hard-coded defaults
- Execute level-specific scripts (MML, Lua, music, load screens) at appropriate times (level entry, game end, restoration)

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `XML_Loader_SDL::ParseFile()`, `ParseDirectory()` ΓÇô file-based entry points
- `XML_DataBlock::ParseData()` ΓÇô memory-based entry point
- `SetupParseTree()`, `ResetAllMMLValues()` ΓÇô initialization and reset
- Factory functions `Color_GetParser()`, `Damage_GetParser()`, `Shapes_GetParser()`, etc.
- Pointer-setter APIs: `Color_SetArray()`, target-setting for damage/shape parsers

**Consumes from other subsystems:**
- Expat XML parser library (C API)
- FileSpecifier, OpenedFile, DirectorySpecifier (file handling)
- Music singleton (FadeOut, PushBackLevelMusic, SeedLevelMusic, ClearLevelMusic)
- OGL_LoadScreen singleton (end-game screen configuration, conditional)
- Lua script loader (LoadLuaScript, conditional)
- Logging system (logAnomaly, anomaly throttling)
- Game engine types: rgb_color, damage_definition, shape_descriptor, color macros

## Runtime Role

- **Engine init:** `SetupParseTree()` constructs the global RootParser hierarchy; `SetupParseTree()` is called once at startup
- **Configuration load:** `XML_Loader_SDL::ParseDirectory()` or `ParseFile()` invoked to load XML configuration files; Expat feeds data chunks to parser, parser stack maintains hierarchy
- **Level load:** `XML_LevelScript` parser identifies level-specific script entries; executes MML, Lua, music, and load screen commands
- **Global scripts:** Pseudo-level parsers (Default, Restore, End) apply configurations across all levels or at special times
- **Error handling:** Errors accumulated during parse; interpretation errors reported via logging with throttle; parse aborted if threshold exceeded
- **Reset:** `ResetAllMMLValues()` restores all MML-configured values to engine defaults

## Notable Implementation Details

- Expat integration via C static callback wrappers; instance state accessed through XML_SetUserData() and cast in callbacks
- Element parser tree is hierarchical; each element parser maintains a children registry (case-insensitive lookup) and a pointer to parent for validation
- Bounded numerical parsing enforces min/max ranges; numerical parsing uses sscanf with field width and format validation
- UTF-8 to ASCII/Mac Roman conversion via external `unicode_to_mac_roman()` function; supports legacy Mac Roman encoding
- Three error levels: read errors (SDK file I/O failures) reported with platform-specific dialogs; XML parse errors reported with line numbers from Expat; interpretation errors accumulated in logging system and throttled to prevent spam
- Singleton pattern with factory functions for each subsystem parser; pointer-setter APIs allow caller-provided storage (e.g., `Color_SetArray(&rgb_array)`)
- Level scripts support conditional execution via pseudo-level identifiers (Default = all levels, Restore = on game restore, End = end-of-game sequence)
- Parsing halts if interpretation error count exceeds threshold; remaining parse aborted gracefully
