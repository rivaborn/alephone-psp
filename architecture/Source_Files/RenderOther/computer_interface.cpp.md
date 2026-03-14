# Source_Files/RenderOther/computer_interface.cpp

## File Purpose
Implements the computer terminal interface system for the game engine, enabling players to interact with in-game terminals that display structured text, graphics, audio, and navigation options. Manages terminal rendering, input handling, state tracking, and serialization across save/load cycles.

## Core Responsibilities
- **Terminal Lifecycle**: Initialize terminal manager, set up per-player terminal state, enter/exit terminal mode
- **Terminal Rendering**: Draw terminal UI (borders, text, pictures), manage clipping regions, apply text styling (bold, italic, color)
- **Text Layout**: Calculate line breaks with word-wrapping, measure text width, determine text bounds for multi-line content
- **State Management**: Track per-player terminal state (current group, line, phase), handle terminal progression through content groups
- **User Input**: Process keyboard input for navigation (next/previous group, page up/down, abort)
- **Terminal Groups**: Manage different group types (logon, success, failure, information, checkpoint, sound, movie, picture, teleport, etc.)
- **Serialization**: Pack/unpack terminal data and player state for save files
- **Terminal Effects**: Handle logon animations, static displays, text encoding/decoding

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `terminal_text_t` | struct | Single terminal's data: flags, lines per page, groupings vector, font changes vector, text buffer |
| `player_terminal_data` | struct | Per-player terminal state: flags, phase, state, current group/line, terminal ID, completion state |
| `terminal_groupings` | struct | Content group: type, permutation, start/length indices, maximum line count |
| `text_face_data` | struct | Font styling: index into text, face flags (bold/italic/underline), color index |
| `font_dimensions` | struct | Font metrics: lines per screen, character width |
| `terminal_key` | struct | Input binding: SDL keycode, bitmask, action flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `map_terminal_text` | `vector<terminal_text_t>` | static | All terminals loaded from current map |
| `player_terminals` | `player_terminal_data*` | static | Array of per-player terminal state (indexed by player) |
| `terminal_keys[]` | `terminal_key[]` | static | SDL keycode-to-action mappings (UP/DOWN/PAGEUP/PAGEDOWN/TAB/ENTER/SPACE/ESC) |
| `current_pixel` | `uint32` | static | Current text color as SDL pixel value |
| `current_style` | `uint16` | static | Current text style flags (normal/bold/italic/underline) |
| `world_pixels`, `draw_surface` | `SDL_Surface*` | extern | Rendering target surfaces |

## Key Functions / Methods

### initialize_terminal_manager
- **Signature**: `void initialize_terminal_manager(void)`
- **Purpose**: Set up terminal system at game startup
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Allocates `player_terminals` array, initializes key mappings on Mac platform
- **Calls**: `new`, `objlist_clear()`
- **Notes**: Called once during initialization; sets up per-player terminal state array

### enter_computer_interface
- **Signature**: `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- **Purpose**: Transition player into terminal mode, display terminal UI
- **Inputs**: Player index, terminal text ID, level completion state
- **Outputs/Return**: None
- **Side effects**: Sets player's terminal state to `_reading_terminal`, plays logon sound, recalculates lines per page if single-player
- **Calls**: `get_player_data()`, `get_indexed_terminal_data()`, `play_object_sound()`, `calculate_lines_per_page()`, `next_terminal_group()`
- **Notes**: Returns early if no terminal data exists in map; calls Lua callback `L_Call_Terminal_Exit()` for prior terminal

### _render_computer_interface
- **Signature**: `void _render_computer_interface(struct view_terminal_data *data)`
- **Purpose**: Render current terminal state to screen
- **Inputs**: Terminal view rectangle (top/left/bottom/right bounds)
- **Outputs/Return**: None
- **Side effects**: Sets clipping rectangle, draws borders/text/pictures based on current group type, clears dirty flag
- **Calls**: `get_indexed_grouping()`, `set_drawing_clip_rectangle()`, `draw_terminal_borders()`, `draw_logon_text()`, `draw_computer_text()`, `present_checkpoint_text()`, `display_picture_with_text()`, `fill_terminal_with_static()`
- **Notes**: Early return if terminal state is `_no_terminal_state` or terminal data not found

### calculate_line
- **Signature**: `static bool calculate_line(char *base_text, short width, short start_index, short text_end_index, short *end_index)`
- **Purpose**: Compute where a line of text breaks given max pixel width
- **Inputs**: Text buffer, line width in pixels, start index, text end index; output pointer for end index
- **Outputs/Return**: `true` if at end of text, `false` if more text follows
- **Side effects**: Modifies `*end_index` with calculated line break point
- **Calls**: `GetInterfaceFont()`, `char_width()`
- **Notes**: Implements word-wrapping by finding last space before width overflow; handles MAC_LINE_END (13) as hard line break

### update_player_for_terminal_mode
- **Signature**: `void update_player_for_terminal_mode(short player_index)`
- **Purpose**: Per-tick update for players in terminal mode (handle timeout transitions)
- **Inputs**: Player index
- **Outputs/Return**: None
- **Side effects**: Decrements phase timer, calls `next_terminal_group()` if timeout expires
- **Calls**: `get_player_terminal_data()`, `get_indexed_terminal_data()`, `next_terminal_group()`
- **Notes**: Phase timer used only for logon/logoff animations; assumes 1 tick per call

### player_in_terminal_mode
- **Signature**: `bool player_in_terminal_mode(short player_index)`
- **Purpose**: Query whether player is currently in terminal interface
- **Inputs**: Player index
- **Outputs/Return**: `true` if player state is `_reading_terminal`, `false` otherwise
- **Side effects**: None
- **Calls**: `get_player_terminal_data()`
- **Notes**: Non-blocking query function

### unpack_map_terminal_data / pack_map_terminal_data
- **Signature**: `void unpack_map_terminal_data(uint8 *p, size_t count)` / `void pack_map_terminal_data(uint8 *p, size_t count)`
- **Purpose**: Deserialize/serialize all terminals from/to binary stream (for map loading/saving)
- **Inputs**: Byte stream pointer, count (unpack: bytes to read; pack: unused)
- **Outputs/Return**: None
- **Side effects**: `unpack` clears `map_terminal_text` vector and populates it; `pack` iterates vector
- **Calls**: `StreamToValue()`, `ValueToStream()`, `StreamToBytes()`, `BytesToStream()`
- **Notes**: Handles groupings, font changes, and text data separately within each terminal's packed format

**Trivial Helpers** (documented briefly):
- `set_text_face()` ΓÇö Apply font style and color from `text_face_data`
- `calculate_lines_per_page()` ΓÇö Determine max lines that fit in terminal bounds
- `get_player_terminal_data()` ΓÇö Bounds-checked accessor for player's terminal state
- `get_indexed_grouping()`, `get_indexed_font_changes()`, `get_text_base()` ΓÇö Vector/buffer accessors with range checks
- `InsetRect()`, `SetRect()`, `OffsetRect()` ΓÇö MacOS-style rectangle manipulation
- Rendering helpers: `draw_terminal_borders()`, `draw_logon_text()`, `draw_computer_text()`, `display_picture()`, `fill_terminal_with_static()` ΓÇö Draw specific UI elements

## Control Flow Notes

**Initialization Phase**:
1. `initialize_terminal_manager()` called once at startup
2. `initialize_player_terminal_info()` called per player
3. Terminal data unpacked from map via `unpack_map_terminal_data()`

**Per-Tick Flow** (during terminal mode):
1. `update_player_for_terminal_mode()` ΓÇö Update timers
2. `update_player_keys_for_terminal()` ΓÇö Process input, may change current group/line
3. `_render_computer_interface()` ΓÇö Draw to screen (called during render phase)

**Terminal Navigation**:
- Player enters via `enter_computer_interface()` ΓåÆ sets state to `_reading_terminal` ΓåÆ calls `next_terminal_group()`
- Input handler `handle_reading_terminal_keys()` advances line or group
- `next_terminal_group()` moves to next group type, handles special transitions (teleport, sound, etc.)
- Player exits when reaching `_end_group` or pressing abort key

**Serialization**: Terminal data is packed before save, unpacked after load, independent of per-player state (which packs separately).

## External Dependencies
- **Includes**: `cseries.h` (common utilities), `FileHandler.h` (I/O), `world.h`/`map.h`/`player.h` (game world types), `screen_drawing.h`/`screen.h` (rendering), `SoundManager.h` (audio), `lua_script.h` (scripting callbacks), `Packing.h` (binary serialization)
- **External Functions**: `_get_font_spec()`, `_get_interface_color()`, `GetInterfaceFont()`, `GetInterfaceStyle()`, `play_object_sound()`, `Sound_TerminalLogon()`, `L_Call_Terminal_Exit()`, `get_player_data()`, `draw_polygon()`, `char_width()`, `text_width()`
- **External Symbols**: `world_pixels`, `draw_surface` (SDL surfaces); `dynamic_world`, `current_player_index` (game state); `game_is_networked` (network flag)
