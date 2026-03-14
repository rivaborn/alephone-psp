# Source_Files/Misc/thread_priority_sdl_dummy.cpp

## File Purpose
Provides a dummy/no-op implementation of thread priority boosting for systems where the underlying platform does not support it. Prints a one-time warning and returns success to maintain API compatibility downstream.

## Core Responsibilities
- Implement `BoostThreadPriority()` as a stub for unsupported platforms
- Print a single warning to alert developers that network performance may suffer
- Return `true` to signal nominal success to the caller

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|--------------------------------|---------|
| `didPrintOutWarning` | `bool` | static (file scope) | Tracks whether warning has been printed; ensures warning prints only once per session |

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Stub implementation for thread priority boosting on unsupported platforms.
- **Inputs:** `inThread` ΓÇô pointer to an SDL thread (unused in this implementation)
- **Outputs/Return:** Always returns `true` (success)
- **Side effects (global state, I/O, alloc):** First call prints warning to stdout; modifies static `didPrintOutWarning` flag
- **Calls (direct calls visible in this file):** `printf()` (from `<stdio.h>`)
- **Notes:** Returns success despite doing nothing, preserving caller expectations. Warning is printed exactly once due to static guard, preventing spam across multiple invocations.

## Control Flow Notes
Likely called during engine initialization or networking setup to attempt thread priority elevation. On unsupported platforms, this stub gracefully degrades by returning success while notifying the user of the limitation.

## External Dependencies
- `thread_priority_sdl.h` ΓÇô declares the `BoostThreadPriority()` function signature
- `<stdio.h>` ΓÇô `printf()` for warning output
- SDL library ΓÇô `SDL_Thread` opaque type (forward-declared in header, not defined in this file)
