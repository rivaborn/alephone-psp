# Source_Files/Input/psp_mouse_sdl.h

## File Purpose
Defines a PSP-to-SDL mouse adapter class that translates PlayStation Portable controller input into SDL mouse events and simulated cursor movement. Supports both digital (button-driven) and analog (stick-driven) mouse control modes with configurable acceleration physics.

## Core Responsibilities
- Map PSP controller buttons to SDL mouse buttons and track button state changes
- Simulate SDL mouse events (button press/release, movement, positioning) from controller input
- Manage mouse position, velocity, and acceleration physics in analog mode
- Support dynamic button binding configuration
- Provide query/setter methods for mouse parameters (acceleration, max velocity, analog threshold)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PSPButtons | enum | Maps all 22 PSP physical buttons (SELECT, START, face buttons, triggers, HOLD, etc.) to indices |
| PSPSDLMouseMode | enum | Selects between MOUSE_ANALOG (stick-driven) and MOUSE_DIGITAL (button-driven) control modes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PSP_BUTTONS_MAP[] | PspCtrlButtons | static | Lookup table converting PSPButtons enum indices to PspCtrlButtons constants |

## Key Functions / Methods

### PSPSDLMouse (constructor)
- Signature: `PSPSDLMouse(PSPSDLMouseMode mode = MOUSE_DIGITAL, float acceleration = DEFAULT_ACC, float vmax = DEFAULT_VMAX, int analog_threshold = DEFAULT_ATHD)`
- Purpose: Initialize mouse handler with specified control mode and physics parameters
- Inputs: Mouse mode, acceleration, max velocity, analog stick threshold
- Outputs/Return: None
- Side effects: Initializes member state (button arrays, position/velocity, SDL event structure)
- Notes: Defaults to digital mode with preset physics constants

### update
- Signature: `void update(void)`
- Purpose: Per-frame input polling and event generation; main entry point for controller-to-mouse translation
- Inputs: Reads current PSP controller state (psp_pad) and button state history (psp_old_buttons)
- Outputs/Return: None (side effects only)
- Side effects: Updates mouse position/velocity, generates SDL events via simulated_event, updates button state tracking
- Calls: update_mouse() and (indirectly) update_mouse_digital/analog
- Notes: Called once per frame by engine

### bind_button / get_button_binding
- Signature: `void bind_button(PspCtrlButtons psp_button, Uint8 mouse_button)` / `PspCtrlButtons get_button_binding(Uint8 mouse_button)`
- Purpose: Configure which PSP button triggers which SDL mouse button
- Inputs: PSP button ID and SDL mouse button ID (0ΓÇô4)
- Outputs/Return: Mapped PSP button (get_button_binding)
- Side effects: Updates mouse_bindings[] lookup table
- Notes: Supports up to 5 mouse buttons and 22 PSP buttons

### Simulation methods (simulate_mouse_button, simulate_mouse_move, simulate_mouse_position)
- Purpose: Allow external code to inject synthetic mouse input (e.g., for testing or menu control)
- Inputs: Button state, delta movement (dx/dy), or absolute position (x/y)
- Outputs/Return: None
- Side effects: Populates simulated_event for SDL consumption
- Notes: Bypass normal controller polling; useful for UI overlays or debugging

### Accessors (get_mouse_x, get_mouse_y, get_mouse_vx, get_mouse_vy, get_mouse_vmax, get_mouse_acceleration, get_analog_threshold, get_mouse_mode)
- Purpose: Query current mouse state and parameters
- Outputs/Return: Respective member variables
- Notes: All inline trivial getters

### Mutators (set_mouse_mode, set_mouse_acceleration, set_mouse_vx, set_mouse_vy, set_mouse_vmax, set_analog_threshold)
- Purpose: Modify mouse behavior at runtime (e.g., adjust acceleration during gameplay)
- Inputs: New parameter value
- Side effects: Update member variable
- Notes: All inline trivial setters

## Control Flow Notes
- **Frame loop**: Engine calls `update()` each frame. Internally, `update()` dispatches to `update_mouse_digital()` or `update_mouse_analog()` depending on `mode`.
- **Analog mode**: Reads analog stick position, applies acceleration/velocity clamping, updates mouse_x/mouse_y.
- **Digital mode**: Reads directional buttons, maps them to velocity adjustments or discrete movement.
- **Button events**: `process_button_event()` fires on state changes (press/release) and populates `simulated_event` for SDL processing.
- **Hybrid input**: External code can inject synthetic input via `simulate_*` methods; these override or supplement controller input.

## External Dependencies
- `#include <pspctrl.h>` ΓÇô PSP controller API (PspCtrlButtons, SceCtrlData types and control reading functions)
- `#include "SDL.h"` ΓÇô SDL library (SDL_Event, Uint8 types; event generation assumed)
- **Defined elsewhere**: PSP_BUTTONS_MAP array definition, update_mouse() / update_mouse_digital() / update_mouse_analog() implementations, process_button_event() / get_button_id() implementations
