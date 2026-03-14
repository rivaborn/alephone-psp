# Source_Files/Misc/thread_priority_sdl_macosx.cpp

## File Purpose
Provides a platform-specific macOS implementation for boosting a thread's priority to its maximum scheduling level. Bridges SDL thread abstractions to POSIX pthread APIs for real-time performance tuning in the Aleph One game engine.

## Core Responsibilities
- Extract native pthread ID from SDL thread handle
- Query current thread scheduling policy and parameters
- Elevate thread priority to system maximum for its scheduling class
- Return success/failure status to caller

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Maximize scheduling priority of a worker thread to improve responsiveness (e.g., audio/input processing).
- **Inputs:** `inThread` ΓÇö SDL thread handle to boost
- **Outputs/Return:** `true` if priority set successfully; `false` if either policy query or priority setting failed
- **Side effects:** Modifies kernel scheduling parameters for the target thread; may require elevated privileges on some systems
- **Calls:** 
  - `SDL_GetThreadID()` ΓÇö retrieve native pthread ID
  - `pthread_getschedparam()` ΓÇö query scheduling policy and current priority bounds
  - `sched_get_priority_max()` ΓÇö lookup maximum priority for policy
  - `pthread_setschedparam()` ΓÇö apply new priority
- **Notes:** 
  - Documented to work on macOS 10.1.0+
  - No error details returned; both policy query and priority-setting failures return `false`
  - Assumes caller (main thread) has permission to modify target thread scheduling

## Control Flow Notes
Intended to be called once during initialization from the main thread to boost performance-critical worker threads (audio, input, physics). No frame/render involvement.

## External Dependencies
- `SDL/SDL_Thread.h` ΓÇö SDL thread abstraction
- `pthread.h` ΓÇö POSIX thread API (`pthread_getschedparam`, `pthread_setschedparam`)
- `sched.h` ΓÇö POSIX scheduling constants and utilities (`sched_get_priority_max`)
