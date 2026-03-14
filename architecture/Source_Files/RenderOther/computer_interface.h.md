# Source_Files/RenderOther/computer_interface.h

## File Purpose
Defines the interface for the in-game computer/terminal display system. Manages player interaction with terminals, including text rendering with formatting, input handling, viewport management, and persistence of terminal state via pack/unpack operations.

## Core Responsibilities
- Initialize and manage the terminal interface system
- Handle player entry into and exit from terminal mode
- Render formatted terminal text with style markup (bold, italic, underline)
- Process player input while interacting with terminals
- Pack and unpack terminal state for save/load operations
- Preprocess and encode terminal text resources from map data
- Manage per-player terminal viewing state (viewport bounds, offsets)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `static_preprocessed_terminal_data` | struct | Binary representation of preprocessed terminal text, containing metadata (length, grouping/font-change counts) and text groups |
| `view_terminal_data` | struct | Viewport state for terminal display (bounds: top/left/bottom/right, vertical scroll offset) |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_terminal_manager
- Signature: `void initialize_terminal_manager(void)`
- Purpose: Initialize the terminal manager system
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes global terminal system state
- Calls: Not inferable from this file

### enter_computer_interface
- Signature: `void enter_computer_interface(short player_index, short text_number, short completion_flag)`
- Purpose: Transition player into terminal mode, displaying specified terminal text
- Inputs: Player index, terminal text ID, completion flag (success/failure/etc.)
- Outputs/Return: None
- Side effects: Changes player mode, loads terminal content
- Calls: Not inferable from this file

### _render_computer_interface
- Signature: `void _render_computer_interface(struct view_terminal_data *data)`
- Purpose: Render the terminal display each frame
- Inputs: Current viewport/window bounds
- Outputs/Return: None
- Side effects: Updates screen, renders text and UI elements
- Calls: Not inferable from this file

### update_player_keys_for_terminal
- Signature: `void update_player_keys_for_terminal(short player_index, uint32 action_flags)`
- Purpose: Process player input (key presses/actions) while in terminal mode
- Inputs: Player index, action flags (converted from keymap via `build_terminal_action_flags`)
- Outputs/Return: None
- Side effects: Modifies terminal viewport state, may trigger text navigation/selection
- Calls: Not inferable from this file

### pack_player_terminal_data / unpack_player_terminal_data
- Signature: `uint8 *pack_player_terminal_data(uint8 *Stream, size_t Count)` / `uint8 *unpack_player_terminal_data(uint8 *Stream, size_t Count)`
- Purpose: Serialize/deserialize per-player terminal state to/from byte stream
- Inputs: Byte stream pointer, count
- Outputs/Return: Updated stream pointer
- Side effects: Modifies stream buffer, updates player terminal state
- Calls: Not inferable from this file

**Trivial helpers summarized under Notes:**
- `build_terminal_action_flags()` ΓÇö converts keymap string to action flags
- `dirty_terminal_view()` ΓÇö marks viewport for redraw
- `abort_terminal_mode()` ΓÇö force exit terminal mode
- `player_in_terminal_mode()` ΓÇö query if player is currently in terminal

## Control Flow Notes
Part of the per-frame game loop: players enter via `enter_computer_interface()`, then each frame calls `update_player_for_terminal_mode()` and `update_player_keys_for_terminal()` for input/state updates, followed by `_render_computer_interface()` for rendering. Exit via `abort_terminal_mode()` or natural completion. Terminal state persists via pack/unpack functions.

## External Dependencies
- **Includes:** Standard C headers (implied by types like `uint8`, `uint32`, `size_t`, `bool`)
- **Game systems:** Player management (indexed by `player_index`), rendering/graphics, sound and movie systems (for terminal content)
- **Conditional code:** Preprocessing functions (text parsing, PICT resources, checkpoints) gated by `PREPROCESSING_CODE` macro (used at map compile-time, not runtime)
