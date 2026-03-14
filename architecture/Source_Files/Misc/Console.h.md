# Source_Files/Misc/Console.h

## File Purpose
Provides a console system for Aleph One game engine with interactive command input, command parsing/execution, and carnage (kill) reporting. Manages a singleton console instance with support for macros, Lua integration, and extensible command registration.

## Core Responsibilities
- **Command Management**: Register/unregister commands and nested command parsers via callback functions
- **Interactive Input**: Handle keyboard input, backspace, clearing, and prompt management
- **Input Callbacks**: Activate/deactivate input mode with user-supplied completion callbacks
- **Macro System**: Register text replacement macros for user convenience
- **Carnage Reporting**: Track and report kill events with customizable kill/suicide messages
- **Lua Integration**: Toggle Lua console support based on user preferences
- **Save/Load State**: Clear saved level metadata

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CommandParser::command_map` | typedef | Maps command strings to callback functions via `boost::function<void(const std::string&)>` |
| `Console::m_macros` | `std::map<std::string, std::string>` | Stores macro name ΓåÆ replacement text mappings |
| `Console::m_carnage_messages` | `std::vector<std::pair<std::string, std::string>>` | Stores kill message templates (on_kill, on_suicide pairs) indexed by projectile type |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Console::m_instance` | `Console*` | static | Singleton instance pointer |

## Key Methods

### CommandParser::register_command (overload 1)
- Signature: `void register_command(std::string command, boost::function<void(const std::string&)> f)`
- Purpose: Register a command with a callback function
- Inputs: command name, callback function accepting the command argument string
- Outputs/Return: void
- Side effects: Modifies `m_commands` map
- Calls: (not visible in header)
- Notes: Allows dynamic command registration; callback is invoked on command execution

### CommandParser::register_command (overload 2)
- Signature: `void register_command(std::string command, const CommandParser& command_parser)`
- Purpose: Register a nested command parser, enabling hierarchical command structures
- Inputs: command name, another CommandParser instance
- Outputs/Return: void
- Side effects: Modifies `m_commands` map
- Calls: (not visible in header)
- Notes: Enables subcommand systems

### CommandParser::parse_and_execute
- Signature: `virtual void parse_and_execute(const std::string& command_string)`
- Purpose: Parse and execute a command string
- Inputs: Raw command string from user
- Outputs/Return: void
- Side effects: May invoke registered callbacks or modify game state
- Calls: (implementation not visible)
- Notes: Virtual; can be overridden by subclasses; execution path depends on registered commands

### Console::instance
- Signature: `static Console* instance()`
- Purpose: Accessor for singleton console instance
- Inputs: none
- Outputs/Return: Pointer to global Console instance (created on first call)
- Side effects: May construct singleton if first call
- Calls: (not visible in header)
- Notes: Thread-safety not specified

### Console::enter / abort / backspace / clear / key
- Purpose: Handle character-by-character keyboard input during active console input
- `enter()`: Confirm and execute input, invoke callback with buffer contents
- `abort()`: Cancel input, invoke callback with empty string
- `backspace()`: Remove last character from buffer
- `clear()`: Erase entire buffer
- `key(const char)`: Add a character to buffer
- Side effects: Modify `m_buffer`, `m_displayBuffer`, possibly invoke `m_callback`
- Notes: No-op if `m_active == false` (except `activate_input`)

### Console::activate_input / deactivate_input
- Signature: `void activate_input(boost::function<void (const std::string&)> callback, const std::string& prompt)`
- Purpose: Enter interactive input mode with a user-supplied completion callback
- Inputs: callback function and prompt string
- Outputs/Return: void (deactivate takes no args)
- Side effects: Set `m_active = true`, store callback and prompt; `deactivate_input` sets `m_active = false` without invoking callback
- Notes: `deactivate_input` is like `abort` but without callback invocation

### Console::set_carnage_message
- Signature: `void set_carnage_message(int16 projectile_type, const std::string& on_kill, const std::string& on_suicide = "")`
- Purpose: Register kill/suicide message templates for a projectile type
- Inputs: projectile type ID, kill message template, optional suicide message (defaults to empty)
- Outputs/Return: void
- Side effects: Store message pair in `m_carnage_messages`
- Notes: Indexed by projectile type; used for displaying contextual kill information

### Console::report_kill
- Signature: `void report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`
- Purpose: Log a kill event for carnage reporting
- Inputs: victim player index, attacker player index, projectile type index
- Outputs/Return: void
- Side effects: May update internal kill statistics; likely logs message based on `m_carnage_messages` entry
- Calls: (implementation not visible)
- Notes: Index `-1` may indicate environmental/suicide kills

## Control Flow Notes
Console operates in two modes:
1. **Inactive**: Commands from calling code via `register_command` / `parse_and_execute`
2. **Active (Interactive)**: User-driven keyboard input via `enter/abort/backspace/key`, culminating in callback invocation

Not directly tied to frame loop; driven by external key handlers and game state (e.g., save/load dialogs, death replays for carnage reporting).

## External Dependencies
- `<string>`, `<map>`, `<boost/function.hpp>`: Standard containers and Boost function objects
- `XML_ElementParser`: For configuration/scripting (header included but usage not visible here)
- `preferences.h`: Accesses `environment_preferences->use_solo_lua` global variable
- `cstypes.h` (via XML_ElementParser): Likely provides `int16`, `uint16`, etc.
