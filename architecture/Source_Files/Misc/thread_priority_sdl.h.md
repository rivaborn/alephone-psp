# Source_Files/Misc/thread_priority_sdl.h

## File Purpose
Header providing a utility function to optimize thread priorities in the Aleph One game engine. Allows boosting worker thread priority or reducing main thread priority to ensure responsive gameplay and audio processing.

## Core Responsibilities
- Declare the thread priority boost interface for SDL-based threads
- Enable performance optimization by prioritizing critical background threads
- Manage priority trade-offs between main and worker threads

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_Thread | (forward decl.) | SDL thread handle; defined elsewhere in SDL library |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Elevate a worker thread's priority substantially, or if unavailable on the platform, reduce the main thread's priority once to create relative priority difference.
- **Inputs:** `SDL_Thread* inThread` ΓÇô pointer to thread to boost
- **Outputs/Return:** `bool` ΓÇô true on success, false if priority adjustment failed
- **Side effects:** Modifies OS-level thread priority; main thread priority reduction is one-time (not cumulative across calls)
- **Calls:** Not inferable from this file (implementation elsewhere)
- **Notes:** Must be called by the main thread to enable main-thread priority reduction. Intended for audio/network processing threads that must not be starved by main game loop.

## Control Flow Notes
Utility function, likely invoked during thread creation (e.g., when spawning audio or network worker threads). Not part of frame/render loop.

## External Dependencies
- `SDL_Thread` (forward declaration; from SDL library)
