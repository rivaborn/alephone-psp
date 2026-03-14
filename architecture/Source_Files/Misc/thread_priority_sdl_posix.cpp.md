# Source_Files/Misc/thread_priority_sdl_posix.cpp

## File Purpose
Implements POSIX thread priority boosting for SDL threads. Allows the main thread to elevate a worker thread's priority to the maximum allowed by its scheduling policy, improving responsiveness for critical threads on POSIX-compliant systems.

## Core Responsibilities
- Boost SDL thread priority to maximum via POSIX scheduling APIs
- Convert SDL thread handles to native pthread_t for kernel manipulation
- Conditionally apply priority scheduling only on systems with `_POSIX_PRIORITY_SCHEDULING` support
- Handle platform-specific SDL header includes (Mac Carbon vs generic)
- Provide error reporting for pthread operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SDL_Thread` | struct (external) | Opaque SDL thread handle passed as input |
| `pthread_t` | typedef (external) | Native POSIX thread identifier |
| `sched_param` | struct (external) | POSIX scheduling parameters (priority field) |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Elevate the specified SDL thread's priority to the maximum allowed by its current scheduling policy.
- **Inputs:** 
  - `inThread` ΓÇô pointer to SDL_Thread to boost
- **Outputs/Return:** 
  - `bool` ΓÇô `true` on success or if POSIX scheduling unavailable; `false` if pthread priority adjustment fails
- **Side effects:** 
  - Modifies kernel thread priority via `pthread_setschedparam()`
  - Only executes if `_POSIX_PRIORITY_SCHEDULING` is defined; otherwise silently succeeds
- **Calls:**
  - `SDL_GetThreadID(inThread)` ΓÇô extract native pthread_t from SDL handle
  - `pthread_getschedparam()` ΓÇô query current scheduling policy and parameters
  - `sched_get_priority_max()` ΓÇô obtain maximum priority for policy
  - `pthread_setschedparam()` ΓÇô apply new priority to thread
- **Notes:** 
  - No validation that thread is still alive; orphaned pointers will fail at `pthread_getschedparam()`
  - Per header comment: intended to be called by main thread; if POSIX scheduling unavailable, main thread priority reduction is skipped
  - Assumes SDL thread exists and is valid

## Control Flow Notes
Utility function called opportunistically (not part of frame loop). Typically invoked during initialization to prioritize worker threads (e.g., networking, audio, I/O threads) ahead of game logic threads.

## External Dependencies
- **SDL:** `SDL_Thread` type, `SDL_GetThreadID()`
- **POSIX threading:** `<pthread.h>` ΓÇô `pthread_t`, `pthread_getschedparam()`, `pthread_setschedparam()`
- **POSIX scheduling:** `<sched.h>` ΓÇô `sched_param`, `sched_get_priority_max()`
- **Platform conditioning:** Detects Mac Carbon (`TARGET_API_MAC_CARBON && __MACH__`) to conditionally include `<SDL/SDL_Thread.h>` vs `<SDL/SDL_thread.h>`
