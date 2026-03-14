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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| LevelScriptCommand | struct | Single command in a script (MML, Music, Movie, Lua, LoadScreen) with resource ID, file spec, size, and positioning data |
| LevelScriptHeader | struct | All commands for one level (or pseudo-level: Default/Restore/End) plus random-order flag |
| XML_LSCommandParser | class | Parses individual script command XML elements and collects attributes into LevelScriptCommand |
| XML_RandomOrderParser | class | Parses "random_order" XML element to control music playback order |
| XML_GeneralLevelScriptParser | class | Base parser for level scripts; manages CurrScriptPtr and creates/finds scripts by level index |
| XML_LevelScriptParser | class | Parser for regular level scripts with "index" attribute |
| XML_SpecialLevelScriptParser | class | Parser for pseudo-level scripts (default, restore, end) with fixed level identifiers |
| XML_EndScreenParser | class | Parses end-of-game screen configuration (resource index and count) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| LevelScripts | vector<LevelScriptHeader> | static | Repository of all parsed level scripts, indexed by level |
| CurrScriptPtr | LevelScriptHeader* | static | Points to the currently-executing script during parsing and runtime |
| MapParentDir | DirectorySpecifier | static | Root directory for map-related files (scripts, movies, music) |
| LuaFound | bool | static (ifdef HAVE_LUA) | Flag indicating whether a Lua script was successfully loaded for current level |
| MovieFile | FileSpecifier | static | File path to the current level's movie (if any) |
| MovieFileExists | bool | static | Whether MovieFile points to an existing file |
| MovieSize | float | static | Relative size for rendering movie (-1 means unset) |
| EndScreenIndex | short | global | Starting resource ID for end-of-game TEXT resources |
| NumEndScreens | short | global | Number of consecutive end screens to display |
| LSXML_Loader | XML_DataBlock | static | XML parser and data block handler for level scripts |
| LSRootParser, LevelScriptSetParser | XML_ElementParser | static | Root and child XML element parsers for level script tree |
| MMLParser, MusicParser, MovieParser, etc. | XML_LSCommandParser/subclass | static | Individual command-type parsers (MML, Music, Movie, Lua, LoadScreen, RandomOrder) |

## Key Functions / Methods

### LoadLevelScripts
- **Signature:** `void LoadLevelScripts(FileSpecifier& MapFile)`
- **Purpose:** Load and parse all level scripts from a map file's TEXT resource 128
- **Inputs:** MapFile ΓÇô file specifier for the map file to extract scripts from
- **Outputs/Return:** None (modifies global LevelScripts, MapParentDir, EndScreenIndex, NumEndScreens)
- **Side effects:** Clears previous LevelScripts (except on first call), resets end-screen defaults, calls XML parser on resource data
- **Calls:** MapFile.ToDirectory(), get_text_resource_from_scenario(), SetupLSParseTree(), LSXML_Loader.ParseData()
- **Notes:** First invocation preserves external scripts; TEXT resource 128 must exist or function returns silently

### RunLevelScript
- **Signature:** `void RunLevelScript(int LevelIndex)`
- **Purpose:** Execute all scripts for a specific level, including MML, music, Lua, and load screens
- **Inputs:** LevelIndex ΓÇô level number to run scripts for
- **Outputs/Return:** None
- **Side effects:** Fades music, clears playlist, resets/reloads MML (except first time), clears load screen, runs Default and level-specific scripts, seeds music playlist
- **Calls:** Music::instance()->FadeOut(), Music::instance()->ClearLevelMusic(), OGL_LoadScreen::instance()->Clear(), ResetAllMMLValues(), LoadBaseMMLScripts(), GeneralRunScript(), Music::instance()->SeedLevelMusic()
- **Notes:** On first call, skips reset to preserve initial state; orchestrates the main per-level initialization sequence

### RunEndScript
- **Signature:** `void RunEndScript()`
- **Purpose:** Execute scripts for end-of-game state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Runs Default and End pseudo-level scripts
- **Calls:** GeneralRunScript()
- **Notes:** Allows end-specific MML and configuration

### RunRestorationScript
- **Signature:** `void RunRestorationScript()`
- **Purpose:** Execute restoration scripts to revert MML changes when returning to menus
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Runs Default and Restore pseudo-level scripts
- **Calls:** GeneralRunScript()
- **Notes:** Used to clean up MML state without reloading the full script set

### GeneralRunScript
- **Signature:** `void GeneralRunScript(int LevelIndex)`
- **Purpose:** Find and execute all commands for a specific level or pseudo-level
- **Inputs:** LevelIndex ΓÇô level or pseudo-level (-1=End, -2=Default, -3=Restore) to execute
- **Outputs/Return:** None
- **Side effects:** Loads resources, parses MML/Lua data, calls Music/OpenGL to apply music and load screens, sets LuaFound flag
- **Calls:** get_text_resource_from_scenario(), LoadLuaScript(), Music::instance()->LevelMusicRandom(), Music::instance()->PushBackLevelMusic(), OGL_LoadScreen::instance()->Set()
- **Notes:** Returns silently if script not found; iterates commands, handling MML, Lua, Music, Movies (data-only), and LoadScreen types; supports conditional compilation

### FindLevelMovie
- **Signature:** `void FindLevelMovie(short index)`
- **Purpose:** Search for a movie associated with a level for playback
- **Inputs:** index ΓÇô level index to search for movie
- **Outputs/Return:** None (modifies MovieFileExists, MovieSize, MovieFile)
- **Side effects:** Searches Default and level-specific scripts; clears MovieSize and MovieFileExists initially
- **Calls:** FindMovieInScript()
- **Notes:** Called before show_movie() to retrieve level movie

### FindMovieInScript
- **Signature:** `void FindMovieInScript(int LevelIndex)`
- **Purpose:** Search a specific script for movie commands
- **Inputs:** LevelIndex ΓÇô level script to search
- **Outputs/Return:** None
- **Side effects:** Sets MovieFile and MovieFileExists if movie found
- **Calls:** None
- **Notes:** Iterates commands; only processes Movie-type commands

### GetLevelMovie
- **Signature:** `FileSpecifier *GetLevelMovie(float& Size)`
- **Purpose:** Retrieve the movie file and size for playback after FindLevelMovie
- **Inputs:** Size ΓÇô reference to float to receive movie size (only updated if valid)
- **Outputs/Return:** Pointer to MovieFile if exists; NULL otherwise
- **Side effects:** Updates Size parameter if MovieSize >= 0
- **Calls:** None
- **Notes:** Companion to FindLevelMovie; called by show_movie()

### SetupLSParseTree
- **Signature:** `static void SetupLSParseTree()`
- **Purpose:** One-time initialization of the XML parser tree for level scripts
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Adds child parsers to LSRootParser and LevelScriptSetParser; sets static WasSetUp flag
- **Calls:** AddChild() on parser instances
- **Notes:** Guard-protected to run only once; called from LoadLevelScripts

### XML_GeneralLevelScriptParser::SetLevel
- **Signature:** `void SetLevel(short Level)`
- **Purpose:** Locate or create a script for a given level and set CurrScriptPtr
- **Inputs:** Level ΓÇô level index
- **Outputs/Return:** None
- **Side effects:** Searches LevelScripts; creates new LevelScriptHeader and appends if not found; sets CurrScriptPtr
- **Calls:** None
- **Notes:** Called during XML parsing to prepare the current script for command insertion

### XML_LSCommandParser::HandleAttribute / Start / End
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)` / `bool Start()` / `bool End()`
- **Purpose:** Parse XML attributes for level script commands (resource/file/size/stretch/progress coords/colors)
- **Inputs:** Tag/Value pairs from XML; Start/End mark element lifecycle
- **Outputs/Return:** true if parsing succeeds; false on unrecognized attributes or missing object
- **Side effects:** Populates Cmd member; Start initializes ObjectWasFound and color array; End appends Cmd to current script
- **Calls:** ReadBoundedInt16Value(), ReadBooleanValueAsBool(), ReadFloatValue(), ReadInt16Value(), Color_SetArray()
- **Notes:** Recognizes: resource (int), file (string), stretch (bool), size (float), progress_top/bottom/left/right (int16)

**Notes (summary of smaller helpers):**
- AddScriptCommands() ΓÇô adds standard parsers (MML, Music, Movie, Lua, LoadScreen, RandomOrder) to a script parser
- XML_RandomOrderParser::HandleAttribute ΓÇô parses "on" bool attribute for music shuffle
- XML_LevelScriptParser::HandleAttribute ΓÇô parses "index" attribute for level number
- XML_SpecialLevelScriptParser ΓÇô inherits SetLevel behavior; used for Default/Restore/End pseudo-levels
- XML_EndScreenParser::HandleAttribute ΓÇô parses "index" and "count" for end screens
- LevelScript_GetParser() / EndLevelScript_GetParser() / DefaultLevelScript_GetParser() / RestoreLevelScript_GetParser() / EndScreens_GetParser() ΓÇô factory functions returning configured parser instances

## Control Flow Notes

**Parse-Time (Map Load):**
LoadLevelScripts() ΓåÆ SetupLSParseTree() ΓåÆ LSXML_Loader.ParseData() ΓåÆ XML parsers (LSRootParser ΓåÆ LevelScriptSetParser ΓåÆ LevelScriptParser/SpecialLevelScript parsers) ΓåÆ XML_LSCommandParser instances ΓåÆ LevelScripts vector populated

**Execution-Time (Level Start):**
RunLevelScript(levelIndex) ΓåÆ Music fade/clear, MML reset ΓåÆ GeneralRunScript(Default) ΓåÆ GeneralRunScript(levelIndex) ΓåÆ foreach command: parse resource/file, execute (MML to RootParser, Lua to LoadLuaScript, Music to PushBackLevelMusic, LoadScreen to OGL_LoadScreen)

**Movie Lookup:**
FindLevelMovie(index) ΓåÆ FindMovieInScript(Default) ΓåÆ FindMovieInScript(index) ΓåÆ GetLevelMovie() returns MovieFile

**Game End:**
RunEndScript() ΓåÆ GeneralRunScript(Default) + GeneralRunScript(End)

**State Restoration:**
RunRestorationScript() ΓåÆ GeneralRunScript(Default) + GeneralRunScript(Restore)

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
