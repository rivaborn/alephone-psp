# Source_Files/XML/ColorParser.cpp
## File Purpose
Parses XML `<color>` elements and populates an external `rgb_color` array with parsed values.
Handles conversion of float color channels [0ΓÇô1] to 16-bit integer format, and optional indexed storage.
Provides a reusable singleton parser instance for XML pipeline integration.

## Core Responsibilities
- Define `XML_ColorParser` class as an XML element parser for the "color" tag
- Parse three color attributes: `red`, `green`, `blue` (float values)
- Optionally parse `index` attribute for array storage position
- Validate all required attributes are present before commit
- Convert and clamp float channel values [0ΓÇô1] to uint16 range [0ΓÇô65535]
- Store validated color into caller-provided array at specified index

## External Dependencies
- **Base class:** `XML_ElementParser` (defined elsewhere; parent for parser state machine)
- **Types:** `rgb_color` (16-bit color struct)
- **Functions (defined elsewhere):** 
  - `StringsEqual(const char*, const char*)` ΓÇô case-insensitive string comparison
  - `ReadBoundedInt16Value(const char*, int16&, int16, int16)` ΓÇô parse bounded int
  - `ReadFloatValue(const char*, float&)` ΓÇô parse float value
  - `PIN(value, min, max)` ΓÇô clamp macro
- **Includes:** `<string.h>`, `ColorParser.h`, `cseries.h`, `XML_ElementParser.h`

# Source_Files/XML/ColorParser.h
## File Purpose
Provides an interface for parsing color elements from XML configuration files. This header defines two public functions that work together to configure color parsing: obtaining a parser instance and setting the target array where parsed color values are stored.

## Core Responsibilities
- Provides factory function to obtain an `XML_ElementParser` specialized for color elements
- Manages configuration of the target color array for parsing operations
- Supports both indexed (array-based) and non-indexed color value modes
- Handles initialization of the color parsing subsystem

## External Dependencies
- `#include "cseries.h"` ΓÇö Core series library (defines/declares `rgb_color` and SDL integration)
- `#include "XML_ElementParser.h"` ΓÇö XML parsing framework base class
- Defined elsewhere: `rgb_color` type, `XML_ElementParser` class implementation

# Source_Files/XML/DamageParser.cpp
## File Purpose
Implements an XML parser for damage element definitions in the Aleph One game engine. Parses damage attributes (type, flags, base, random, scale) from XML and populates a damage_definition structure. Provides a singleton parser interface for reuse across multiple damage definitions.

## Core Responsibilities
- Parse XML damage element attributes and validate their values
- Convert floating-point scale values to fixed-point representation
- Enforce bounds on damage type (0 to NUMBER_OF_DAMAGE_TYPES-1) and flags (0ΓÇô1)
- Provide a getter for the damage parser instance
- Set the target damage_definition pointer before parsing begins

## External Dependencies
- **Includes:** `cseries.h` (standard utilities), `DamageParser.h`, `<string.h>`, `<limits.h>`
- **Defined elsewhere:** 
  - `XML_ElementParser` base class
  - `damage_definition` struct (from map.h)
  - `NUMBER_OF_DAMAGE_TYPES`, `FIXED_ONE` macros
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadBoundedNumericalValue()`, `UnrecognizedTag()` functions
  - `SHRT_MIN`, `SHRT_MAX` from `<limits.h>`

# Source_Files/XML/DamageParser.h
## File Purpose
Interface for parsing XML "damage" elements into `damage_definition` structures. Provides factory and target-setting functions for an XML parser specialized in damage configuration.

## Core Responsibilities
- Expose factory function to obtain a damage XML parser instance
- Define mechanism to specify the target `damage_definition` object for parsing results
- Abstract XML parsing details for damage-related game data

## External Dependencies
- **map.h** ΓÇô `damage_definition` struct, damage type/flag enums
- **XML_ElementParser.h** ΓÇô base parser class that this module wraps

# Source_Files/XML/ShapesParser.cpp
## File Purpose
Parses XML `<shape>` elements for the Aleph One game engine, extracting shape descriptor properties (collection, color lookup table, sequence/frame indices) and composing them into a single `shape_descriptor` value.

## Core Responsibilities
- Implement an XML element parser for shape definitions
- Validate and parse shape attributes (`coll`, `clut`, `seq`, `frame`)
- Enforce required attributes and acceptable bounds
- Compose validated attributes into a final `shape_descriptor` via builder macros
- Provide singleton parser instance and pointer-setter API for reuse across multiple XML elements

## External Dependencies
- **`XML_ElementParser`** (defined elsewhere): Base class for element-specific XML parsers.
- **`shape_descriptor`** (defined elsewhere): Game engine type representing a shape reference.
- **Parsing utilities** (defined elsewhere): `ReadBoundedUInt16Value()`, `StringsEqual()`, `UnrecognizedTag()`, `AttribsMissing()`, macros `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `UNONE`.
- **Constants** (defined elsewhere): `MAXIMUM_COLLECTIONS`, `MAXIMUM_CLUTS_PER_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`.

# Source_Files/XML/ShapesParser.h
## File Purpose
This header provides a factory interface for parsing shape descriptor elements from XML configuration files. It decouples XML parsing of shape data from the shape descriptor system by offering a getter for configured parsers and a setter for the destination pointer where parsed values should be stored.

## Core Responsibilities
- Provide access to an `XML_ElementParser` instance specialized for shape element parsing
- Manage destination pointers for parsed shape descriptor values
- Support optional "NONE" values in parsed shape fields
- Abstract the shape parsing logic from callers (implementation in corresponding .cpp file)

## External Dependencies
- **Includes:**
  - `shape_descriptors.h` ΓÇö defines `shape_descriptor` type and collection/shape macros
  - `XML_ElementParser.h` ΓÇö base class for all XML element parsers

- **External symbols used:**
  - `shape_descriptor` (typedef, defined elsewhere)
  - `XML_ElementParser` (class, defined elsewhere)

# Source_Files/XML/XML_Configure.cpp
## File Purpose
Implementation of the XML_Configure class, which orchestrates XML file parsing for the Aleph One game engine. Uses the Expat parser library to read XML files and delegates semantic interpretation to a tree of XML_ElementParser objects.

## Core Responsibilities
- Manages Expat parser creation, setup, and lifecycle
- Implements Expat callbacks (static wrappers) that delegate to instance methods
- Navigates and maintains a tree of XML element parsers (CurrentElement)
- Handles XML element lifecycle: start, attribute processing, character data, end
- Accumulates and reports XML parsing and semantic interpretation errors
- Orchestrates the read-parse loop, feeding data chunks to Expat until completion

## External Dependencies
- **Standard C**: `<stdio.h>`, `<stdarg.h>` (vsprintf, va_list, va_start, va_end)
- **Expat library**: XML_Parser, XML_ParserCreate, XML_SetUserData, XML_SetElementHandler, XML_SetCharacterDataHandler, XML_Parse, XML_ParserFree, XML_ErrorString, XML_GetErrorCode, XML_GetCurrentLineNumber (all imported from "expat.h")
- **Game engine**: "cseries.h" (utilities); "XML_Configure.h" (class definition)
- **XML_ElementParser** (external class): FindChild, Parent, GetName, Start, HandleAttribute, AttributesDone, End, HandleString, NameMatch, ErrorString field (defined elsewhere)

# Source_Files/XML/XML_Configure.h
## File Purpose
Abstract base class that orchestrates XML parsing for Marathon engine configuration. Wraps the Expat XML parser library and delegates element-level interpretation to a hierarchical `XML_ElementParser` structure. Subclasses provide file I/O and error handling policy.

## Core Responsibilities
- Manage the Expat parser lifecycle and coordinate XML parsing flow
- Provide callback routing from Expat's C API to C++ instance methods
- Maintain a stack of active element parsers for nested XML structures
- Track and report parsing, XML syntax, and interpretation errors
- Define abstract interface for subclasses to supply data and handle errors

## External Dependencies
- **expat.h**: Low-level C XML parsing library; provides `XML_Parser` type and callback registration functions
- **XML_ElementParser.h**: Hierarchical element parser; interprets parsed XML structure semantically
- Standard C: `<stdlib.h>`, `<cstdarg>` (inferred for variadic `ComposeInterpretError`)

# Source_Files/XML/XML_DataBlock.cpp
## File Purpose
Implements the XML_DataBlock class for parsing XML contained in memory buffers. Handles error reporting at three levels (read, parse, interpretation) with platform-specific error dialogs and graceful logging. Part of the Aleph One game engine's XML configuration system.

## Core Responsibilities
- Provides XML data blocks to the parser via GetData()
- Reports fatal read errors with platform-specific error dialogs (Mac/SDL)
- Reports XML parsing errors with line numbers for debugging
- Reports interpretation errors via logging with throttling to avoid spam
- Requests parsing abortion when error threshold is exceeded
- Maintains source name for debugging context

## External Dependencies
- **cseries.h**: Cross-platform utilities (`csprintf`, `psprintf`, `SimpleAlert`, `ParamText`, `Alert`, `ExitToShell`)
- **XML_DataBlock.h**: Class declaration; parent class `XML_Configure`
- **Logging.h**: Logging system (`logAnomaly1`, `logAnomaly`, `GetCurrentLogger`)
- **stdio.h (implicit)**: `fprintf` for SDL stderr output
- **stdlib.h (implicit)**: `exit()` for SDL termination
- Inherited: `GetNumInterpretErrors()`, `DoParse()` from `XML_Configure`

# Source_Files/XML/XML_DataBlock.h
## File Purpose
Defines `XML_DataBlock`, a concrete parser that loads and parses XML data from memory buffers. It inherits from `XML_Configure` to handle XML parsing for data blocks in the Aleph One engine's configuration system.

## Core Responsibilities
- Provide a data-block-specific XML parsing interface by implementing the `XML_Configure` contract
- Accept XML source data from memory buffers via `ParseData()`
- Optionally track the source location/name of parsed XML for debugging
- Implement error reporting hooks (read, parse, interpretation errors)
- Coordinate with the Expat parser to parse buffered XML content

## External Dependencies
- **`XML_Configure.h`** ΓÇö parent class; defines the core parsing interface and Expat integration
- **`expat.h`** (via `XML_Configure.h`) ΓÇö the Expat XML parser library
- **`XML_ElementParser.h`** (via `XML_Configure.h`) ΓÇö element parsing infrastructure

# Source_Files/XML/XML_ElementParser.cpp
## File Purpose
Implementation of the `XML_ElementParser` base class, which provides utilities for parsing XML elements, attributes, and text content. Handles numerical and boolean value parsing, element hierarchy management, and UTF-8 string conversion with Mac Roman encoding support.

## Core Responsibilities
- Parse typed values (int16, uint16, int32, uint32, float, boolean) from XML attribute strings with validation
- Manage a hierarchy of child XML element parsers with duplicate prevention
- Perform case-insensitive string matching for XML element and attribute names
- Convert UTF-8 encoded strings to ASCII/Mac Roman for compatibility
- Generate and track descriptive error messages for parsing failures
- Support bounded numerical parsing with min/max range validation

## External Dependencies
- **cseries.h** ΓÇö engine umbrella header (includes standard types, SDL, MacOS emulation)
- **cstypes.h** ΓÇö fixed-width integer types (int16, uint16, int32, uint32, uint8)
- **XML_ElementParser.h** ΓÇö class declaration, template definitions
- **\<string.h\>** ΓÇö strlen, strcpy
- **\<ctype.h\>** ΓÇö toupper
- **\<vector\>** ΓÇö STL container for children list
- **\<stdio.h\>** ΓÇö sscanf for numerical parsing
- **unicode_to_mac_roman()** ΓÇö external function (defined elsewhere) for character conversion

# Source_Files/XML/XML_ElementParser.h
## File Purpose
Base class framework for hierarchical XML element parsing. Provides parsing infrastructure for attributes, child elements, and text content. Designed to be subclassed for specific XML element types with custom parsing logic.

## Core Responsibilities
- Manage XML element hierarchy (parent/child relationships)
- Parse and validate numerical values (with optional bounds checking)
- Parse boolean and floating-point values from strings
- Provide lifecycle hooks (Start, End, AttributesDone, HandleString)
- Track parsing errors via ErrorString
- Manage child element registry with case-insensitive lookup
- Support UTF-8 to ASCII string conversion

## External Dependencies
- `<vector>` (STL) ΓÇö child element container
- `<stdio.h>` ΓÇö sscanf for numerical parsing
- `cstypes.h` ΓÇö cross-platform integer type definitions (int16, uint16, int32, uint32)
- `XML_GetBooleanValue()` ΓÇö extern function for boolean string parsing
- `StringsEqual()` ΓÇö case-insensitive string comparison (defined in this file)
- `DeUTF8()`, `DeUTF8_Pas()`, `DeUTF8_C()` ΓÇö UTF-8 conversion utilities (defined in this file)


# Source_Files/XML/XML_LevelScript.cpp
## File Purpose

Implements XML script parsing and execution for Marathon level configurations. Loads scripts from map file resources that specify per-level commands (MML definitions, music, movies, Lua code, load screens), and executes these scripts at appropriate times (level entry, game end, restoration).

## Core Responsibilities

- Load level scripts from map file resource 128 (TEXT) and parse XML into a global script database
- Execute level-specific commands when levels are entered, including MML, Lua, music, movies, and load screens
- Manage pseudo-level scripts (Default, Restore, End) that apply globally or at special times
- Parse XML attributes for script commands and levels from map files
- Search for and retrieve movies associated with levels for playback
- Configure end-of-game screens via XML attributes
- Maintain global state for current script execution and movie lookup

## External Dependencies

**Headers:**
- `<vector>` ΓÇô STL for LevelScripts, Commands, playlist
- `cseries.h` ΓÇô core types, strings (StringsEqual, sprintf)
- `shell.h` ΓÇô shell interface, file specs
- `game_wad.h` ΓÇô game file handling
- `Music.h` ΓÇô Music singleton (FadeOut, PushBackLevelMusic, SeedLevelMusic, LevelMusicRandom, ClearLevelMusic)
- `ColorParser.h` ΓÇô Color_GetParser(), Color_SetArray()
- `XML_DataBlock.h`, `XML_ElementParser` ΓÇô XML parsing framework
- `XML_LevelScript.h` ΓÇô public interface
- `XML_ParseTreeRoot.h` ΓÇô RootParser, SetupParseTree(), ResetAllMMLValues()
- `Random.h` ΓÇô random number support
- `images.h` ΓÇô get_text_resource_from_scenario()
- `lua_script.h` (conditional) ΓÇô LoadLuaScript()
- `OGL_LoadScreen.h` (conditional) ΓÇô OGL_LoadScreen singleton, Set()

**External Functions:**
- LoadBaseMMLScripts(), ResetAllMMLValues() ΓÇô MML infrastructure
- get_text_resource_from_scenario(id, resource) ΓÇô retrieve TEXT from scenario
- LoadLuaScript(data, len) ΓÇô parse/execute Lua
- Music::instance()->* ΓÇô music control (FadeOut, PushBack, Seed, Random, Clear)
- OGL_LoadScreen::instance()->* ΓÇô load screen control
- Read*Value() functions ΓÇô XML value parsers (Int16, Float, Boolean)
- FileSpecifier, DirectorySpecifier ΓÇô file/path handling
- LoadedResource ΓÇô resource data wrapper

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

## External Dependencies
- **FileHandler.h** ΓÇö provides `FileSpecifier` class for file/path abstraction
- **XML parser** ΓÇö `XML_ElementParser` (forward declared; implementation defined elsewhere)
- **Pfhortran scripting engine** ΓÇö referenced in comments; integration points not visible in this header
- **MML system** ΓÇö supports Markup Macro Language execution; no declarations in this header

# Source_Files/XML/XML_Loader_SDL.cpp
## File Purpose
SDL-based XML file parser for the Aleph One game engine. Loads and parses XML configuration files from disk, manages file I/O, and reports parsing errors with context (filename, line number, error count limits).

## Core Responsibilities
- Load XML file data into memory buffers and expose to parser
- Parse individual XML files via `ParseFile()`
- Batch-parse all XML files in a directory via `ParseDirectory()`, filtering backup files (~) and Lua scripts (.lua)
- Report read errors, parse errors, and interpretation errors with context
- Enforce error reporting limits and request parse abort if threshold exceeded
- Track current filename for error messages

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `OpenedFile` classes for file I/O abstraction
- **XML_Configure.h**: Base class; defines `DoParse()`, `GetNumInterpretErrors()` (inheritance, not shown)
- **cseries.h**: Common game engine headers and macros
- **boost/algorithm/string/predicate.hpp**: `ends_with()` for Lua script filtering
- **stdio.h, vector, algorithm**: Standard C++ libraries

# Source_Files/XML/XML_Loader_SDL.h
## File Purpose
SDL-based XML parser for the Marathon/Aleph One game engine. Extends the generic `XML_Configure` class to load and parse XML configuration files from disk, with file-aware error reporting.

## Core Responsibilities
- Load XML file data from disk via `FileSpecifier` objects
- Scan directories for XML files to parse
- Provide parsed data buffers to the parent `XML_Configure` parser
- Report parsing/reading errors with source filename context
- Manage memory for loaded XML data

## External Dependencies
- `XML_Configure.h` (base class providing DoParse, parser lifecycle, expat integration)
- `XML_Configure::Buffer`, `BufLen`, `LastOne` (inherited protected fields filled by GetData)
- `FileSpecifier` (file system abstraction, defined elsewhere)
- expat parser (included via XML_Configure.h)

# Source_Files/XML/XML_MakeRoot.cpp
## File Purpose
Constructs and initializes the root of the XML parser tree for the Marathon game engine. It defines the global root and "marathon" element parsers, registers all subsystem XML parsers as children, and provides functions to set up the complete parse tree and reset MML-configured values to defaults.

## Core Responsibilities
- Define the absolute XML document root element
- Create the canonical "marathon" element as the top-level game configuration root
- Register ~30 subsystem parsers as children of the marathon element
- Build the complete hierarchical XML parse tree during engine initialization
- Provide a reset mechanism to restore all MML-configured values to hard-coded defaults

## External Dependencies
- **XML infrastructure:** `XML_ElementParser`, `XML_ParseTreeRoot.h` (root declarations; class definition elsewhere)
- **Subsystem parsers (all `*_GetParser()` functions defined in respective modules):**
  - Game logic: `Player`, `Items`, `Weapons`, `Monsters`, `Scenery`, `Platforms`, `Liquids`
  - Rendering/UI: `Interface`, `MotionSensor`, `OverheadMap`, `AnimatedTextures`, `Landscapes`, `OpenGL`, `SW_Texture_Extras`
  - Input/Output: `Keyboard`, `Console`, `SoundManager`, `Logging`
  - Configuration: `TextStrings`, `PlayerName`, `DynamicLimits`, `Scenario`, `ExternalDefaultLevelScript`
  - Conditionals: `Theme_GetParser()` (SDL only)
- **Engine headers:** `world.h`, `interface.h`, `game_window.h`, `vbl.h`, `shell.h` (platform/window management)

# Source_Files/XML/XML_ParseTreeRoot.h
## File Purpose
Header file declaring the root of the XML parser tree hierarchy. Provides the absolute root element that anchors all valid XML document parsers and exposes initialization/reset routines for the entire parsing structure.

## Core Responsibilities
- Export the global `RootParser` object (root of the parse tree)
- Declare `SetupParseTree()` to initialize and configure the complete parser hierarchy
- Declare `ResetAllMMLValues()` to reset all parsed values back to hard-coded defaults
- Serve as the single entry point for XML parsing initialization

## External Dependencies
- **`XML_ElementParser.h`** ΓÇô base class for all element parsers; provides the hierarchical parser framework
- **`cstypes.h`** (transitively via XML_ElementParser.h) ΓÇô likely defines platform-specific integer types


