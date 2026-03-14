# Source_Files/Misc/Console.cpp

## File Purpose

Console utilities for the Aleph One game engine, providing a command-line interface for in-game console input. Implements command parsing, macro expansion, and carnage message reporting (kill notifications in networked games). Integrates with the engine's XML configuration system.

## Core Responsibilities

- Singleton `Console` class for managing real-time console input and display
- Command registration and parsing with `CommandParser`
- Text macro expansion (input/output substitution)
- Carnage message managementΓÇöformatted kill notifications with player name substitution
- Save command infrastructure (level export)
- XML configuration parsing for macros, carnage messages, and console settings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Console` | class | Singleton managing console state, input buffer, callbacks, and command dispatch |
| `CommandParser` | class | Hierarchical command dispatcher; base for `Console` |
| `save_level` | functor/struct | Operator for handling `.save level` command; manages level export with filename caching |
| `XML_CarnageMessageParser` | class | Parses `<carnage_message>` XML elements; registers kill message templates |
| `XML_MacroParser` | class | Parses `<macro>` XML elements; registers text replacement macros |
| `XML_ConsoleParser` | class | Parses `<console>` XML element; configures Lua console flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Console::m_instance` | `Console*` | static (class member) | Singleton instance pointer |
| `last_level` | `std::string` | file-static | Caches the last level filename saved via `.save level` command; persists across console sessions |
| `CarnageMessageParser` | `XML_CarnageMessageParser` | file-static | Singleton XML parser for carnage message definitions |
| `MacroParser` | `XML_MacroParser` | file-static | Singleton XML parser for macro definitions |
| `ConsoleParser` | `XML_ConsoleParser` | file-static | Singleton XML parser for console configuration |
| `game_is_networked` | `bool` (external) | extern | Global flag indicating if the game is in network mode; gated on carnage reporting |

## Key Functions / Methods

### Console::instance()
- **Signature:** `static Console* instance()`
- **Purpose:** Lazy-initialize and return singleton console instance
- **Inputs:** None
- **Outputs/Return:** Pointer to `Console` singleton; creates new instance if `m_instance` is null
- **Side effects (global state, I/O, alloc):** Allocates console on first call
- **Calls:** Constructor `Console()`
- **Notes:** Classic lazy-initialized singleton; not thread-safe

### Console::enter()
- **Signature:** `void enter()`
- **Purpose:** Process the accumulated input buffer and execute it as a command or pass to callback
- **Inputs:** `m_buffer` (internal input string)
- **Outputs/Return:** None; side effects modify state
- **Side effects (global state, I/O, alloc):** Macro expansion (`.macro_name args` ΓåÆ template substitution); command parsing and execution; invokes callback; clears buffers; disables SDL key repeat and Unicode input
- **Calls:** `split()`, `lowercase()`, `parse_and_execute()`, callback function, `SDL_EnableKeyRepeat()`, `SDL_EnableUNICODE()`
- **Notes:** Macros are expanded first (prefix `.` with spaces), then commands (also `.` prefix) are parsed. If no command match, callback is invoked with the original buffer content. Callbacks are cleared after execution.

### Console::abort()
- **Signature:** `void abort()`
- **Purpose:** Cancel console input without processing; invoke callback with empty string
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Calls callback with empty string; clears buffers and state; disables SDL input modes
- **Calls:** Callback function, `SDL_EnableKeyRepeat()`, `SDL_EnableUNICODE()`
- **Notes:** Logs anomaly if no callback was set

### Console::key(char c)
- **Signature:** `void key(const char c)`
- **Purpose:** Add a character to the input buffer and update the display buffer
- **Inputs:** `c` ΓÇö the character to append
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Updates `m_buffer` and `m_displayBuffer` (replaces cursor `_` with character and new `_`)
- **Calls:** None
- **Notes:** Called for each keystroke; does not validate input

### Console::activate_input(callback, prompt)
- **Signature:** `void activate_input(boost::function<void (const std::string&)> callback, const std::string& prompt)`
- **Purpose:** Activate console input mode; set up callback and display prompt
- **Inputs:** `callback` ΓÇö function to invoke when input is confirmed; `prompt` ΓÇö label to display (e.g., "Command >")
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Sets `m_callback`, clears buffers, initializes display with prompt and cursor, enables SDL key repeat and Unicode input, sets `m_active = true`
- **Calls:** `SDL_EnableKeyRepeat()`, `SDL_EnableUNICODE()`
- **Notes:** Asserts if console is already active

### Console::register_macro(input, output)
- **Signature:** `void register_macro(std::string input, std::string output)`
- **Purpose:** Register a text replacement macro (e.g., `input="foo"` ΓåÆ `output="bar"`)
- **Inputs:** `input` ΓÇö macro name (case-insensitive); `output` ΓÇö replacement text
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Modifies `m_macros` map
- **Calls:** `lowercase()`
- **Notes:** Macros are applied in `enter()` when input starts with `.`

### Console::set_carnage_message(projectile_type, on_kill, on_suicide)
- **Signature:** `void set_carnage_message(int16 projectile_type, const std::string& on_kill, const std::string& on_suicide = "")`
- **Purpose:** Register a template message to display when a player is killed or commits suicide with a given projectile type
- **Inputs:** `projectile_type` ΓÇö projectile enum index; `on_kill` ΓÇö message format (contains `%player%` and `%aggressor%` placeholders); `on_suicide` ΓÇö suicide variant
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Stores message pair in `m_carnage_messages[projectile_type]`; sets `m_carnage_messages_exist = true`
- **Calls:** None
- **Notes:** Messages are indexed by projectile type for O(1) lookup in `report_kill()`

### Console::report_kill(player_index, aggressor_player_index, projectile_index)
- **Signature:** `void report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`
- **Purpose:** Display a formatted kill message in networked games
- **Inputs:** `player_index` ΓÇö victim; `aggressor_player_index` ΓÇö killer (or victim if suicide); `projectile_index` ΓÇö weapon used
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Calls `screen_printf()` to display formatted message on HUD
- **Calls:** `get_projectile_data()`, `get_player_data()`, `replace_first()`, `screen_printf()`, `NetAllowCarnageMessages()`
- **Notes:** Returns early if not networked, carnage messages disabled, or no template exists. Substitutes `%player%` and `%aggressor%` with player names; order of substitution matters if both appear and positions overlap.

### CommandParser::parse_and_execute(command_string)
- **Signature:** `void parse_and_execute(const std::string& command_string)`
- **Purpose:** Split input into command name and args; look up and invoke handler
- **Inputs:** `command_string` ΓÇö full command line (e.g., `"save level mymap"`)
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Executes registered function or nested parser
- **Calls:** `split()`, `lowercase()`, callback/nested parser
- **Notes:** Case-insensitive command lookup; passes remainder as single arg string to handler

### Console::register_save_commands()
- **Signature:** `void register_save_commands()`
- **Purpose:** Set up the `.save level` command infrastructure
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Registers a nested `CommandParser` under `"save"` command; that parser handles `"level"` subcommand
- **Calls:** `CommandParser::register_command()`
- **Notes:** Called from `Console` constructor; demonstrates hierarchical command registration pattern

### Console_GetParser()
- **Signature:** `XML_ElementParser* Console_GetParser()`
- **Purpose:** Provide root XML parser for console configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to static `ConsoleParser` root; has `MacroParser` and `CarnageMessageParser` as children
- **Side effects (global state, I/O, alloc):** Modifies parser child list (adds children to `ConsoleParser`)
- **Calls:** `ConsoleParser.AddChild()`
- **Notes:** Called during XML configuration load; parsers are static singletons shared across calls

## Control Flow Notes

**Console Input Lifecycle:**
1. Game code calls `Console::activate_input(callback, prompt)` to request user input
2. Input event handlers call `Console::key(c)` for each keystroke, `Console::backspace()` to delete, or `Console::clear()` to reset
3. When user presses Enter, game calls `Console::enter()`:
   - Macro expansion (if input starts with `.`)
   - Command dispatch (if still starts with `.` after macro expansion)
   - Otherwise, invoke registered callback with full buffer
4. Callback or command handler processes the string; console deactivates

**Carnage Message Flow (in networked games):**
1. When a projectile causes player death, damage system calls `Console::report_kill()`
2. Function looks up projectile type ΓåÆ carnage message template
3. Substitutes player names into template
4. Calls `screen_printf()` to display on HUD

Not inferable: Exact timing of render/update cycle; lifecycle hooks not apparent in this file.

## External Dependencies

- **Notable includes / imports / using directives:**
  - `cseries.h` ΓÇö Aleph One core types (int16, _fixed, macros)
  - `Console.h` ΓÇö Class declaration
  - `Logging.h` ΓÇö `logAnomaly()` for error reporting
  - `network.h` ΓÇö `game_is_networked`, `NetAllowCarnageMessages()`, `NetAllowSavingLevel()`
  - `player.h` ΓÇö `get_player_data()`, player data types
  - `projectiles.h` ΓÇö `get_projectile_data()`, projectile types
  - `shell.h` ΓÇö `screen_printf()`, `FileSpecifier`
  - `FileHandler.h` ΓÇö `FileSpecifier`, `export_level()`
  - `game_wad.h` ΓÇö (included, purpose not evident from this file)
  - `boost/bind.hpp`, `boost/function.hpp` ΓÇö Functor/callback support
  - `boost/algorithm/string/predicate.hpp` ΓÇö `ends_with()`
  - `<string>`, `<map>` ΓÇö Standard containers

- **External symbols used but not defined here:**
  - `game_is_networked` (bool) ΓÇö defined elsewhere
  - `get_projectile_data()` ΓÇö returns projectile metadata
  - `get_player_data()` ΓÇö returns player name and other data
  - `screen_printf()` ΓÇö HUD output
  - `NetAllowCarnageMessages()`, `NetAllowSavingLevel()` ΓÇö network policy checks
  - `export_level()` ΓÇö file I/O for level saving
  - `mac_roman_to_utf8()`, `utf8_to_mac_roman()` ΓÇö character encoding
  - XML parser base class `XML_ElementParser` ΓÇö inherited by the three parser classes
  - `FileSpecifier` ΓÇö file path abstraction
  - `static_world` ΓÇö global level/world state (extern)
