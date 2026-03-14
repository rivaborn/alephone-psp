# Source_Files/Misc/thread_priority_sdl_win32.cpp

## File Purpose
Windows-specific implementation for prioritizing SDL threads, primarily for time-critical network operations. Provides adaptive fallback strategies across Windows versions (Win98 through WinXP+), degrading gracefully when APIs are unavailable.

## Core Responsibilities
- Boost SDL thread priority to time-critical levels (TIME_CRITICAL ΓåÆ HIGHEST ΓåÆ ABOVE_NORMAL)
- Dynamically load and invoke the OpenThread API for WinXP/Win2000+ systems
- Fall back to direct thread handle manipulation for Win98 compatibility
- Reduce main thread priority as a last-resort compensation mechanism
- Emit diagnostic warnings when operations fail

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OpenThreadPtrT | typedef | Function pointer type: `HANDLE (WINAPI*)(DWORD, BOOL, DWORD)` for OpenThread API |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| isMainThreadPriorityReduced | bool | static (function-local in TryToReduceMainThreadPriority) | Guard flag ensuring main thread priority is reduced at most once |

## Key Functions / Methods

### TryToReduceMainThreadPriority
- Signature: `static bool TryToReduceMainThreadPriority()`
- Purpose: Reduce main thread priority to BELOW_NORMAL as a fallback when boosting worker threads fails
- Inputs: None
- Outputs/Return: `true` if reduction succeeded or already applied; `false` if SetThreadPriority fails
- Side effects: Modifies static local state; calls Windows SetThreadPriority
- Calls: GetCurrentThread, SetThreadPriority
- Notes: IdempotentΓÇöreturns `true` on all subsequent calls once the flag is set, regardless of initial success

### BoostThreadPriority
- Signature: `bool BoostThreadPriority(SDL_Thread* inThread)`
- Purpose: Attempt to elevate an SDL thread's priority for time-critical work (e.g., network I/O)
- Inputs: `inThread` (SDL_Thread*)ΓÇötarget thread to boost
- Outputs/Return: `true` if any priority level was successfully set; `false` if all strategies fail
- Side effects: Calls Windows module/API loading; attempts three SetThreadPriority levels; may invoke TryToReduceMainThreadPriority; prints warnings to stdout on failure
- Calls: GetModuleHandle("KERNEL32"), GetProcAddress, SDL_GetThreadID, dynamically-loaded OpenThread, SetThreadPriority (├ù3), CloseHandle, FreeLibrary
- Notes: 
  - **Cascading fallback**: Modern API (OpenThread, Win2000+) ΓåÆ Legacy direct handle (Win98) ΓåÆ Main thread priority reduction
  - **Priority cascade**: Tries TIME_CRITICAL, then HIGHEST, then ABOVE_NORMAL; stops on first success
  - **Handle casting**: Uses `reinterpret_cast<HANDLE>` on SDL_GetThreadID result for Win98 fallback (undocumented; risky but necessary for old Windows)
  - Prints warnings instead of silent failure; network performance may degrade if all attempts fail

## Control Flow Notes
Initialization/setup phase: called asynchronously when network or background threads are created. Not part of frame loop. Works independently of main update/render cycle. Provides priority adjustment outside the standard engine lifecycle.

## External Dependencies
- **Windows API**: windows.h (GetModuleHandle, GetCurrentThread, GetProcAddress, OpenThread, SetThreadPriority, CloseHandle, FreeLibrary)
- **SDL**: SDL_thread.h (SDL_Thread opaque type, SDL_GetThreadID)
- **Standard C**: stdio.h (printf)
- **Local header**: thread_priority_sdl.h (declares BoostThreadPriority extern)
