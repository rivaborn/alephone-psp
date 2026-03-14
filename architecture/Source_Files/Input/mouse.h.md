# Source_Files/Input/mouse.h

## File Purpose
Header file declaring the mouse input interface for the Aleph One game engine (Marathon-derived). Provides functions to initialize, poll, and manage mouse input state, with optional SDL-based mouse button-to-keypress mapping for platforms using SDL.

## Core Responsibilities
- Lifecycle management: initialize and shutdown mouse input for a given device type
- Poll mouse state and extract action flags, yaw/pitch deltas, and velocity changes
- Maintain and recenter mouse cursor position during gameplay
- (SDL) Map mouse buttons to keyboard events for menu/UI interaction
- (SDL) Handle mouse scroll wheel input

## Key Types / Data Structures
None defined in this file.

| External Type | Kind | Purpose |
|--------------|------|---------|
| `uint32` | typedef | Bitfield for action flags (button/action state) |
| `_fixed` | typedef | Fixed-point arithmetic for rotation and velocity deltas |
| `Uint8` | typedef (SDL) | Byte array for SDL keymap |

## Global / File-Static State
None.

## Key Functions / Methods

### enter_mouse
- Signature: `void enter_mouse(short type)`
- Purpose: Initialize mouse input handling for the given device type
- Inputs: `type` ΓÇô device type identifier
- Outputs/Return: None
- Side effects: Initializes mouse input state; may set up SDL event handlers
- Calls: Not inferable from header
- Notes: Paired with `exit_mouse()`

### test_mouse
- Signature: `void test_mouse(short type, uint32 *action_flags, _fixed *delta_yaw, _fixed *delta_pitch, _fixed *delta_velocity)`
- Purpose: Poll mouse state and extract input deltas for frame processing
- Inputs: `type` ΓÇô device type; output pointers for flags and deltas
- Outputs/Return: Populates `action_flags` (button state), `delta_yaw`, `delta_pitch`, `delta_velocity` (look/movement)
- Side effects: Updates internal mouse tracking state
- Calls: Not inferable
- Notes: Called once per frame to gather input; uses fixed-point for precision

### exit_mouse, mouse_idle, recenter_mouse
- Purpose: Shutdown (exit), suspend (idle), and recenter mouse cursor
- Notes: `recenter_mouse()` takes no arguments; likely called after polling to reset position for next frame

### mouse_buttons_become_keypresses (SDL only)
- Signature: `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- Purpose: Synthesize keyboard events from mouse button state
- Inputs: Pointer to SDL keymap array
- Side effects: Modifies keymap in-place
- Notes: Enables mouse button 1ΓÇô8 to trigger keyboard events; uses pseudo-keysym base `SDLK_BASE_MOUSE_BUTTON (65)`

### mouse_scroll (SDL only)
- Signature: `void mouse_scroll(bool up)`
- Purpose: Handle mouse wheel scroll input
- Inputs: `up` ΓÇô true for scroll up, false for scroll down
- Notes: Likely triggers menu/UI navigation or gameplay actions

## Control Flow Notes
Part of the frame input pipeline. `enter_mouse()` called during engine initialization; `test_mouse()` polled during frame update to gather deltas; `exit_mouse()` called during shutdown. `mouse_idle()` used to suspend input during pause/menu states. SDL-specific functions bridge mouse input into the engine's keypress system for compatibility with UI code expecting keyboard events.

## External Dependencies
- **SDL** (conditional): `Uint8` (SDL keymap type)
- **Aleph One internal**: `_fixed` (fixed-point type), action flag system
- **Standard C**: `short`, `uint32`, `bool`
