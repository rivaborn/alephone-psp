# Source_Files/Input/ISp_Support.h
## File Purpose
Header declaring the public interface for InputSprocket support in the Marathon/Aleph One game engine. InputSprocket (ISp) is a legacy macOS input management API. This file provides functions to initialize, configure, and manage input devices during gameplay.

## Core Responsibilities
- Initialize and shut down the InputSprocket system
- Start/stop input event processing during gameplay sessions
- Query active input device type (keyboard vs. other devices)
- Enumerate and test available input devices
- Configure game-specific input control mappings for InputSprocket

## External Dependencies
- InputSprocket API (legacy macOS input framework) ΓÇö defined elsewhere

# Source_Files/Input/mouse.h
## File Purpose
Header file declaring the mouse input interface for the Aleph One game engine (Marathon-derived). Provides functions to initialize, poll, and manage mouse input state, with optional SDL-based mouse button-to-keypress mapping for platforms using SDL.

## Core Responsibilities
- Lifecycle management: initialize and shutdown mouse input for a given device type
- Poll mouse state and extract action flags, yaw/pitch deltas, and velocity changes
- Maintain and recenter mouse cursor position during gameplay
- (SDL) Map mouse buttons to keyboard events for menu/UI interaction
- (SDL) Handle mouse scroll wheel input

## External Dependencies
- **SDL** (conditional): `Uint8` (SDL keymap type)
- **Aleph One internal**: `_fixed` (fixed-point type), action flag system
- **Standard C**: `short`, `uint32`, `bool`

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

## External Dependencies
- **SDL:** `SDL_WM_GrabInput()`, `SDL_EventState()`, `SDL_GetVideoSurface()`, `SDL_WarpMouse()`, `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_ShowCursor()`, `SDL_PumpEvents()`, `SDL_BUTTON()`, `SDL_MOUSEMOTION`, `SDL_GRAB_ON/OFF`, `SDL_PRESSED/RELEASED`, `Uint8`
- **cseries.h:** `_fixed` type, `FIXED_FRACTIONAL_BITS`, `FIXED_ONE`, `PIN()` macro
- **preferences.h:** `input_preferences` global (sensitivity, inversion, acceleration flags)
- **player.h:** action flag bit constants (`_cycle_weapons_forward`, `_cycle_weapons_backward`)

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

## External Dependencies
- **PSP SDK:** `pspctrl.h` ΓÇô `sceCtrlReadBufferPositive()`, `sceCtrlSetSamplingCycle()`, `sceCtrlSetSamplingMode()`, `SceCtrlData`, `PspCtrlButtons` constants
- **SDL 1.2:** `SDL.h` ΓÇô `SDL_Event`, `SDL_PushEvent()`, `SDL_GetVideoSurface()`, `SDL_GetError()`, `SDL_WarpMouse()`, button/event constants
- **C Standard Library:** `cstdio` (printf), `cstring` (memset), `cmath` (fabs)

# Source_Files/Input/psp_mouse_sdl.h
## File Purpose
Defines a PSP-to-SDL mouse adapter class that translates PlayStation Portable controller input into SDL mouse events and simulated cursor movement. Supports both digital (button-driven) and analog (stick-driven) mouse control modes with configurable acceleration physics.

## Core Responsibilities
- Map PSP controller buttons to SDL mouse buttons and track button state changes
- Simulate SDL mouse events (button press/release, movement, positioning) from controller input
- Manage mouse position, velocity, and acceleration physics in analog mode
- Support dynamic button binding configuration
- Provide query/setter methods for mouse parameters (acceleration, max velocity, analog threshold)

## External Dependencies
- `#include <pspctrl.h>` ΓÇô PSP controller API (PspCtrlButtons, SceCtrlData types and control reading functions)
- `#include "SDL.h"` ΓÇô SDL library (SDL_Event, Uint8 types; event generation assumed)
- **Defined elsewhere**: PSP_BUTTONS_MAP array definition, update_mouse() / update_mouse_digital() / update_mouse_analog() implementations, process_button_event() / get_button_id() implementations


