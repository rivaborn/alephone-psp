# Source_Files/Input/mouse_sdl.cpp

## File Purpose
SDL-specific implementation of in-game mouse input handling. Captures mouse movement and button state, calculates yaw/pitch/velocity deltas for game control, and manages mouse pointer visibility. Provides a frame-to-frame snapshot mechanism for input integration.

## Core Responsibilities
- Initialize/shutdown mouse grabbing and event handling (`enter_mouse`, `exit_mouse`)
- Calculate mouse movement deltas with configurable sensitivity and acceleration (`mouse_idle`)
- Return accumulated input deltas and state to game logic (`test_mouse`)
- Recenter mouse pointer after screen size changes (`recenter_mouse`)
- Convert mouse button presses into simulated keypress events (`mouse_buttons_become_keypresses`)
- Manage cursor visibility (`hide_cursor`, `show_cursor`)
- Query current mouse position and button state
- Process scrollwheel input for weapon cycling

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mouse_active` | bool | static | Tracks whether mouse input is active in-game |
| `button_mask` | uint8 | static | Bitmask of which mouse buttons are enabled (debouncing) |
| `center_x`, `center_y` | int | static | Screen center coordinates for delta calculation |
| `snapshot_delta_yaw` | _fixed | static | Accumulated yaw rotation delta (x-axis) |
| `snapshot_delta_pitch` | _fixed | static | Accumulated pitch rotation delta (y-axis) |
| `snapshot_delta_velocity` | _fixed | static | Accumulated forward/backward velocity delta |
| `snapshot_delta_scrollwheel` | _fixed | static | Accumulated scrollwheel input for weapon cycling |

## Key Functions / Methods

### enter_mouse
- **Signature:** `void enter_mouse(short type)`
- **Purpose:** Initialize in-game mouse handling; activate mouse grab and configure SDL event state.
- **Inputs:** `type` ΓÇö input mode (ignored if `_keyboard_or_game_pad`; otherwise enables mouse)
- **Outputs/Return:** None
- **Side effects:** Sets `mouse_active = true`, grabs mouse cursor, disables mouse-motion events in SDL, recenters mouse, clears all delta accumulators and button mask
- **Calls:** `SDL_WM_GrabInput()`, `SDL_EventState()`, `recenter_mouse()`
- **Notes:** Button mask is cleared to prevent stale button states from GUI interactions from firing shots on entry

### exit_mouse
- **Signature:** `void exit_mouse(short type)`
- **Purpose:** Shutdown in-game mouse handling; release mouse grab and restore normal cursor behavior.
- **Inputs:** `type` ΓÇö input mode (ignored if `_keyboard_or_game_pad`)
- **Outputs/Return:** None
- **Side effects:** Sets `mouse_active = false`, releases mouse cursor grab, re-enables mouse-motion events
- **Calls:** `SDL_WM_GrabInput()`, `SDL_EventState()`

### recenter_mouse
- **Signature:** `void recenter_mouse(void)`
- **Purpose:** Recalculate screen center and warp mouse there (called on screen resize or mode change).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Queries video surface dimensions, updates `center_x`/`center_y`, warps cursor to center
- **Calls:** `SDL_GetVideoSurface()`, `SDL_WarpMouse()`
- **Notes:** Only executes if `mouse_active` is true

### mouse_idle
- **Signature:** `void mouse_idle(short type)`
- **Purpose:** Sample current mouse position, compute movement deltas, apply sensitivity/acceleration, and store in snapshots.
- **Inputs:** `type` ΓÇö input mode (`_mouse_yaw_pitch` ΓåÆ yaw+pitch; otherwise yaw+velocity)
- **Outputs/Return:** None
- **Side effects:** Queries mouse position, warps cursor back to center, updates `snapshot_delta_*` accumulators, tracks `last_tick_count` for delta timing
- **Calls:** `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_WarpMouse()`
- **Notes:** 
  - Skips if elapsed ticks < 1 (avoid division by zero or extreme values)
  - X delta = yaw; Y delta = pitch (if `_mouse_yaw_pitch`) or velocity
  - Applies per-axis sensitivity scaling from `input_preferences`
  - Optional nonlinear mouse acceleration: pins to `┬▒FIXED_ONE/2`, squares magnitude, shifts right 14 bits
  - Linear mode: just pins to `┬▒FIXED_ONE/2` and shifts right 1 bit
  - Inverts Y if `_inputmod_invert_mouse` flag set

### test_mouse
- **Signature:** `void test_mouse(short type, uint32 *flags, _fixed *delta_yaw, _fixed *delta_pitch, _fixed *delta_velocity)`
- **Purpose:** Return accumulated mouse state deltas to the game engine and clear snapshots.
- **Inputs:** `type` ΓÇö input mode (unused here); output pointers for deltas and flags
- **Outputs/Return:** Writes to `*delta_yaw`, `*delta_pitch`, `*delta_velocity`, `*flags`
- **Side effects:** Clears `snapshot_delta_*` accumulators; updates `flags` with `_cycle_weapons_forward` / `_cycle_weapons_backward` if scrollwheel delta is non-zero
- **Calls:** None
- **Notes:** If mouse not active, output deltas are assigned 0 (not cleared via pointer dereferenceΓÇö**bug**: should dereference pointers)

### mouse_buttons_become_keypresses
- **Signature:** `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- **Purpose:** Convert mouse button states into simulated SDL keypress events.
- **Inputs:** `ioKeyMap` ΓÇö SDL keymap array to update
- **Outputs/Return:** Modifies `ioKeyMap` in-place
- **Side effects:** Updates `button_mask` to track which buttons have been released (debouncing); sets keymap entries for pseudo-keys `SDLK_BASE_MOUSE_BUTTON + [0..7]`
- **Calls:** `SDL_GetMouseState()`
- **Notes:** Uses `button_mask` to debounce: a button must be released at least once before it can fire again; masked against current button state

### hide_cursor / show_cursor
- **Signature:** `void hide_cursor(void)` / `void show_cursor(void)`
- **Purpose:** Toggle cursor visibility.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets SDL cursor display state
- **Calls:** `SDL_ShowCursor()`

### get_mouse_position
- **Signature:** `void get_mouse_position(short *x, short *y)`
- **Purpose:** Query current absolute mouse position.
- **Inputs:** Output pointers for X and Y
- **Outputs/Return:** Writes to `*x`, `*y`
- **Side effects:** None
- **Calls:** `SDL_GetMouseState()`

### mouse_still_down
- **Signature:** `bool mouse_still_down(void)`
- **Purpose:** Check if left mouse button is currently held down.
- **Inputs:** None
- **Outputs/Return:** `true` if left button pressed
- **Side effects:** Pumps SDL event queue
- **Calls:** `SDL_PumpEvents()`, `SDL_GetMouseState()`

### mouse_scroll
- **Signature:** `void mouse_scroll(bool up)`
- **Purpose:** Accumulate scrollwheel input for weapon cycling.
- **Inputs:** `up` ΓÇö true if scrolling up, false for down
- **Outputs/Return:** None
- **Side effects:** Increments or decrements `snapshot_delta_scrollwheel`
- **Calls:** None

## Control Flow Notes
**Frame lifecycle** (inferred from function design):
1. Game calls `enter_mouse()` when player enters a level
2. Each game tick: `mouse_idle()` samples SDL mouse state and computes deltas
3. Game calls `test_mouse()` to retrieve accumulated deltas and consume them
4. On exit or pause: `exit_mouse()` releases mouse grab

**Input path:** Raw SDL mouse position ΓåÆ sensitivity/acceleration scaling ΓåÆ fixed-point deltas ΓåÆ snapshots ΓåÆ game physics

## External Dependencies
- **SDL:** `SDL_WM_GrabInput()`, `SDL_EventState()`, `SDL_GetVideoSurface()`, `SDL_WarpMouse()`, `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_ShowCursor()`, `SDL_PumpEvents()`, `SDL_BUTTON()`, `SDL_MOUSEMOTION`, `SDL_GRAB_ON/OFF`, `SDL_PRESSED/RELEASED`, `Uint8`
- **cseries.h:** `_fixed` type, `FIXED_FRACTIONAL_BITS`, `FIXED_ONE`, `PIN()` macro
- **preferences.h:** `input_preferences` global (sensitivity, inversion, acceleration flags)
- **player.h:** action flag bit constants (`_cycle_weapons_forward`, `_cycle_weapons_backward`)
