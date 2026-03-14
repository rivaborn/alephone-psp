# Source_Files/Input/psp_mouse_sdl.cpp

## File Purpose
Implements a PSP (PlayStation Portable) controller-to-mouse adapter for SDL applications. Converts PSP buttons and analog stick input into simulated SDL mouse events and position updates, supporting both digital (button-driven) and analog (stick-driven) control modes.

## Core Responsibilities
- Read PSP controller state via `sceCtrlReadBufferPositive()` and detect button state changes
- Dispatch button events to SDL via `SDL_PushEvent()`
- Simulate mouse movement with physics (acceleration, velocity capping, deceleration)
- Constrain cursor position to video surface bounds and zero velocity at edges (prevent "sticking")
- Support two movement modes: digital (button-based with acceleration) and analog (direct stick mapping)
- Maintain button-to-mouse-button binding table for remapping

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PSPSDLMouse` | Class | Main controller-to-mouse adapter (defined in header) |
| `PSPSDLMouseMode` | Enum | Movement mode selector: `MOUSE_ANALOG`, `MOUSE_DIGITAL` |
| `PSPButtons` | Enum | Enumeration of 22 PSP button identifiers |
| `PSP_BUTTONS_MAP[]` | Static array | Maps enum indices to PSP SDK button constants |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PSP_BUTTONS_MAP[]` | `const PspCtrlButtons[]` | Class-static | Maps 22 button enum positions to PSP SDK constants; indices align with `PSPButtons` enum |

## Key Functions / Methods

### PSPSDLMouse (constructor)
- **Signature:** `PSPSDLMouse(PSPSDLMouseMode mode = MOUSE_DIGITAL, float acceleration = DEFAULT_ACC, float vmax = DEFAULT_VMAX, int analog_threshold = DEFAULT_ATHD)`
- **Purpose:** Initialize controller-to-mouse adapter with mode, physics parameters, and PSP SDK sampling
- **Inputs:** Movement mode, acceleration per frame, max velocity, analog stick deadzone threshold
- **Outputs/Return:** None
- **Side effects:** Initializes all member variables to zero; calls `sceCtrlSetSamplingCycle(0)` and `sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG)` to configure PSP hardware sampling
- **Calls:** `set_mouse_mode()`, `set_analog_threshold()`, `simulate_mouse_position()`, `set_mouse_vx()`, `set_mouse_vy()`, `set_mouse_vmax()`, `set_mouse_acceleration()`, `memset()`
- **Notes:** Zeros button binding table; sets `psp_old_buttons = 0` to detect first frame changes

### update
- **Signature:** `void update()`
- **Purpose:** Main per-frame callback; read PSP controller, detect button changes, dispatch events, update mouse position
- **Inputs:** None (reads `psp_pad` from PSP SDK)
- **Outputs/Return:** None
- **Side effects:** Calls `sceCtrlReadBufferPositive()` to poll controller; may push SDL events and warp cursor
- **Calls:** `sceCtrlReadBufferPositive()`, `process_button_event()`, `update_mouse()`
- **Notes:** Optimized to skip processing if button state hasn't changed (`psp_pad.Buttons == psp_old_buttons`); iterates all 22 buttons checking bitwise state

### update_mouse_digital
- **Signature:** `void update_mouse_digital(void)`
- **Purpose:** Apply digital (button-driven) movement with frame-by-frame acceleration/deceleration physics
- **Inputs:** None (reads `buttons_state[PSPB_UP/DOWN/LEFT/RIGHT]`)
- **Outputs/Return:** None
- **Side effects:** Modifies `mouse_vx`, `mouse_vy`, calls `simulate_mouse_move()`, warps cursor
- **Calls:** `get_mouse_vy()`, `set_mouse_vy()`, `get_mouse_vx()`, `set_mouse_vx()`, `get_mouse_acceleration()`, `get_mouse_vmax()`, `simulate_mouse_move()`, `fabs()`
- **Notes:** Accelerates velocity when directional buttons pressed; decelerates on release; snaps velocity to zero when magnitude is below acceleration threshold (prevents sub-pixel sliding); clamps max velocity with sign preservation; applies deceleration per-axis independently

### update_mouse_analog
- **Signature:** `void update_mouse_analog(void)`
- **Purpose:** Map analog stick position directly to mouse velocity with deadzone filtering
- **Inputs:** None (reads `psp_pad.Lx`, `psp_pad.Ly`)
- **Outputs/Return:** None
- **Side effects:** Sets `mouse_vx`, `mouse_vy` directly proportional to stick offset; calls `simulate_mouse_move()`
- **Calls:** `get_analog_threshold()`, `set_mouse_vx()`, `set_mouse_vy()`, `get_mouse_vmax()`, `simulate_mouse_move()`, `fabs()`
- **Notes:** Subtracts 127 (center) from stick values (signed range ┬▒127); zeros velocity if magnitude below threshold; scales remaining stick movement to `[0, vmax]` range

### simulate_mouse_position
- **Signature:** `void simulate_mouse_position(float x, float y)`
- **Purpose:** Update simulated cursor position and enforce boundary constraints
- **Inputs:** Desired x, y coordinates
- **Outputs/Return:** None (cursor warped via `SDL_WarpMouse()`)
- **Side effects:** Updates `mouse_x`, `mouse_y`; retrieves video surface dimensions via `SDL_GetVideoSurface()`; zeros `mouse_vx`/`mouse_vy` if position clamped; calls `SDL_WarpMouse()`
- **Calls:** `SDL_GetVideoSurface()`, `SDL_GetError()`, `printf()`, `SDL_WarpMouse()`
- **Notes:** Clamps position to `[0, w-1]` ├ù `[0, h-1]`; zeros velocity on boundary to prevent sticking; logs error if video surface unavailable

### simulate_mouse_button
- **Signature:** `void simulate_mouse_button(Uint8 button, bool state)`
- **Purpose:** Create and push SDL mouse button down/up event
- **Inputs:** SDL button constant (`SDL_BUTTON_LEFT`, etc.), true for down, false for up
- **Outputs/Return:** None
- **Side effects:** Fills `simulated_event` union with button event data; pushes to SDL event queue via `SDL_PushEvent()`
- **Calls:** `get_mouse_x()`, `get_mouse_y()`, `SDL_PushEvent()`
- **Notes:** Sets event type, button, state, and current mouse position in one call; button position is from `simulate_mouse_position()` state

### process_button_event
- **Signature:** `void process_button_event(PspCtrlButtons button, bool state)`
- **Purpose:** Translate PSP button state change into mouse button events via binding table lookup
- **Inputs:** PSP button code, new state (pressed/released)
- **Outputs/Return:** None
- **Side effects:** Calls `simulate_mouse_button()` for each matching binding
- **Calls:** `get_button_binding()`, `simulate_mouse_button()`
- **Notes:** Linear search across all 5 mouse button slots; unbind buttons produce no events

### bind_button / get_button_binding
- **Signature:** `void bind_button(PspCtrlButtons psp_button, Uint8 mouse_button)` / `PspCtrlButtons get_button_binding(Uint8 mouse_button)`
- **Purpose:** Map PSP button to SDL mouse button and retrieve mapping
- **Inputs:** PSP button constant, mouse button index (0ΓÇô4)
- **Outputs/Return:** Returns PSP button code (or 0 if unbound/out-of-range)
- **Side effects:** `bind_button()` writes `mouse_bindings[mouse_button]`; both bounds-check against `MAX_MOUSE_BUTTONS`
- **Calls:** None (simple array access)

---

## Control Flow Notes
- **Init (constructor):** Hardware configured for analog sampling; state cleared; defaults applied
- **Per-frame (update):** PSP controller polled ΓåÆ button changes detected ΓåÆ events dispatched ΓåÆ physics applied ΓåÆ cursor warped
- **Movement:** Mode-dependent: digital accumulates velocity with frame-based accel/decel; analog maps stick directly with deadzone
- **Output:** SDL events queued; cursor position synchronized via `SDL_WarpMouse()`

## External Dependencies
- **PSP SDK:** `pspctrl.h` ΓÇô `sceCtrlReadBufferPositive()`, `sceCtrlSetSamplingCycle()`, `sceCtrlSetSamplingMode()`, `SceCtrlData`, `PspCtrlButtons` constants
- **SDL 1.2:** `SDL.h` ΓÇô `SDL_Event`, `SDL_PushEvent()`, `SDL_GetVideoSurface()`, `SDL_GetError()`, `SDL_WarpMouse()`, button/event constants
- **C Standard Library:** `cstdio` (printf), `cstring` (memset), `cmath` (fabs)
