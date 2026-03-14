# Subsystem Overview

## Purpose
The Input subsystem manages player control input across multiple platforms (desktop SDL, PSP controller) by providing a unified event model. It captures mouse/controller state, calculates movement deltas with physics, and integrates input into the game loop through SDL's event system.

## Key Files
| File | Role |
|------|------|
| `mouse.h` | Cross-platform mouse interface; declares init, shutdown, poll, and state query functions |
| `mouse_sdl.cpp` | SDL implementation of mouse input handling; captures movement deltas, sensitivity, and button-to-keypress mapping |
| `psp_mouse_sdl.h` | PSP-specific adapter class; defines controller-to-SDL mouse translation interface |
| `psp_mouse_sdl.cpp` | PSP implementation; reads PSP controller state and dispatches SDL mouse events with physics simulation |
| `ISp_Support.h` | Legacy InputSprocket (macOS) device enumeration and configuration (unused on PSP) |

## Core Responsibilities
- Initialize and shut down input device handling (`enter_mouse`, `exit_mouse`)
- Poll input state and extract movement deltas (yaw, pitch, velocity) with configurable sensitivity and acceleration
- Map hardware input (PSP controller buttons, SDL mouse buttons) to game action flags
- Simulate mouse events from PSP controller input (digital button-driven or analog stick-driven modes)
- Manage cursor position, visibility, and recentering during gameplay
- Convert mouse buttons and scroll wheel input to keyboard events for menu interaction
- Constrain cursor to video surface bounds and apply velocity physics (acceleration, deceleration, max velocity caps)

## Key Interfaces & Data Flow
**Exposes to other subsystems:**
- `test_mouse()` ??? returns accumulated input deltas and action flags per frame
- `recenter_mouse()` ??? resets cursor position after screen changes
- `mouse_buttons_become_keypresses()` ??? translates mouse button presses into keyboard events
- Mouse state queries (position, button state) and cursor visibility control

**Consumes from other subsystems:**
- `input_preferences` global (sensitivity, inversion, acceleration settings)
- `player.h` action flag constants (`_cycle_weapons_forward`, etc.)
- SDL event injection points and video surface dimensions
- PSP controller state via `sceCtrlReadBufferPositive()` (PSP only)

## Runtime Role
- **Init:** `enter_mouse()` initializes mouse grabbing, cursor visibility, and PSP controller sampling mode/cycle
- **Frame:** `test_mouse()` / `update_mouse_*()` poll controller/mouse state, calculate deltas with physics, and push SDL events into the event queue
- **Shutdown:** `exit_mouse()` releases mouse grab and restores cursor visibility

## Notable Implementation Details
- **PSP adapter pattern:** `psp_mouse_sdl.cpp` reads raw PSP controller (`SceCtrlData`) and synthesizes SDL mouse events, avoiding modification of mainline SDL code
- **Physics simulation:** Analog stick input uses acceleration and velocity capping; cursor velocity decelerates to zero at surface edges to prevent "sticking"
- **Dual-mode input:** PSP supports digital (button-driven with acceleration) and analog (direct stick mapping) movement modes
- **Button remapping:** PSP_BUTTONS_MAP array allows runtime configuration of controller-to-mouse-button bindings
- **Platform abstraction:** SDL 1.2 provides a common mouse/event interface; PSP-specific code is gated behind preprocessor conditionals and an adapter class rather than inline ifdef blocks
