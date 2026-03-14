# Source_Files/CSeries/mytm_sdl.cpp

## File Purpose
Provides SDL-based Time Manager emulation for periodic task scheduling. Implements drift-free periodic callbacks using SDL threads, approximating classic Mac OS 9 Time Manager behavior for the Aleph One game engine (especially for networking code). Tasks execute with mutual exclusion and priority boosting to prevent starvation.

## Core Responsibilities
- Initialize and manage a global mutex for synchronizing TMTask execution
- Create and manage periodic callback threads with configurable periods
- Implement drift-free scheduling via cumulative drift tracking across resets
- Handle task lifecycle: setup, execution, reset, and removal
- Support task cleanup and zombie thread collection
- (DEBUG) Track profiling metrics: call counts, timing, drift min/max, late deadlines
- Expose mutex lock/unlock for external code (e.g., packet listening thread)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | struct | Housekeeping structure for each periodic task; holds thread pointer, period, callback function, volatiles for coordination (mKeepRunning, mIsRunning, mResetTime), and optional profiling data |
| `myTMTask_profile` | struct (DEBUG only) | Profiling metrics: start/finish times, call counts, drift bounds, late-call and reset statistics |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sTMTaskMutex` | SDL_mutex* | static | Mutex ensuring only one TMTask runs at a time; initialized by `mytm_initialize()` |
| `sOutstandingTasks` | vector<myTMTaskPtr> | static | Vector of active tasks; used for cleanup and resource tracking |

## Key Functions / Methods

### mytm_initialize
- **Signature:** `void mytm_initialize()`
- **Purpose:** One-time initialization of the global TMTask mutex
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates SDL mutex and stores in `sTMTaskMutex`; logs warning/anomaly if creation fails or already called
- **Calls:** `SDL_CreateMutex()`, logging macros
- **Notes:** No matching destructor provided; relies on process exit

### take_mytm_mutex / release_mytm_mutex
- **Signature:** `bool take_mytm_mutex()` / `bool release_mytm_mutex()`
- **Purpose:** Acquire and release the global task mutex
- **Inputs:** None
- **Outputs/Return:** `bool` success (true if lock/unlock succeeded)
- **Side effects:** Calls `SDL_LockMutex()` / `SDL_UnlockMutex()` on `sTMTaskMutex`; logs anomaly on failure
- **Calls:** SDL mutex operations, logging
- **Notes:** Logging in these functions is noted as "potentially a Bad Idea" since logging is not thread-safe, but useful for diagnostics

### thread_loop (static)
- **Signature:** `static int thread_loop(void* inData)` (SDL thread function)
- **Purpose:** Main loop executed by each task's thread; implements drift-free periodic execution
- **Inputs:** `inData` ΓÇö pointer to `myTMTask` structure for this thread
- **Outputs/Return:** Always `0` (SDL thread convention)
- **Side effects:** Runs callback function periodically, modifies task's `mKeepRunning` and `mIsRunning` volatiles, updates profiling data (DEBUG), holds mutex while calling callback
- **Calls:** `SDL_GetTicks()`, `SDL_Delay()`, `take_mytm_mutex()`, `release_mytm_mutex()`, user callback function
- **Notes:** Implements drift tracking (cumulative offset from ideal schedule). Resets on `mResetTime > 0`. Checks for early termination between delays. Small race condition acknowledged: callback may be called once after `myTMRemove()` if preempted before checking `mKeepRunning`

### myTMSetup / myXTMSetup
- **Signature:** `myTMTaskPtr myTMSetup(long time, bool (*func)(void))` / `myTMTaskPtr myXTMSetup(long time, bool (*func)(void))`
- **Purpose:** Create and schedule a periodic task; `myXTMSetup` implements drift correction, `myTMSetup` calls it
- **Inputs:** `time` ΓÇö period in milliseconds; `func` ΓÇö callback function pointer
- **Outputs/Return:** Pointer to newly allocated `myTMTask` 
- **Side effects:** Allocates new `myTMTask`, creates SDL thread, adds to `sOutstandingTasks`, boosts thread priority
- **Calls:** `SDL_CreateThread()`, `BoostThreadPriority()`, `vector::push_back()`
- **Notes:** Callback returns bool; if false, task auto-terminates on next iteration

### myTMRemove
- **Signature:** `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- **Purpose:** Signal a task to stop executing (async, non-blocking)
- **Inputs:** `task` ΓÇö pointer to task to remove
- **Outputs/Return:** Always `NULL`
- **Side effects:** Sets `task->mKeepRunning = false` (volatile write); thread exits on next iteration
- **Calls:** None
- **Notes:** Does not wait for thread to exit (unlike `myTMCleanup` with `inWaitForFinishers=true`)

### myTMReset
- **Signature:** `void myTMReset(myTMTaskPtr task)`
- **Purpose:** Reset a task's periodic delay to original value; resynchronizes to current time
- **Inputs:** `task` ΓÇö pointer to task to reset
- **Outputs/Return:** None
- **Side effects:** If thread is running, sets `mResetTime` to current tick; if thread has exited, waits for it and creates a new thread with same callback/period
- **Calls:** `SDL_GetTicks()`, `SDL_WaitThread()`, `SDL_CreateThread()`, `BoostThreadPriority()`, logging (DEBUG)
- **Notes:** Zombie thread cleanup happens here; small race condition if thread exits between `mIsRunning` check and re-creation

### myTMCleanup
- **Signature:** `void myTMCleanup(bool inWaitForFinishers)`
- **Purpose:** Reclaim stopped tasks: remove from vector, wait for threads, free memory
- **Inputs:** `inWaitForFinishers` ΓÇö if true, clean up running tasks; if false, only collect stopped tasks
- **Outputs/Return:** None
- **Side effects:** Iterates over `sOutstandingTasks`, erases stopped/finished tasks, calls `SDL_WaitThread()` and `delete` on each
- **Calls:** `SDL_WaitThread()`, `vector::erase()`, optional `myTMDumpProfile()` (DEBUG)
- **Notes:** Uses `erase(i--)` idiom to maintain valid iterator; only called at non-time-critical moments

### myTMDumpProfile (DEBUG only)
- **Signature:** `void myTMDumpProfile(myTMTask* inTask)`
- **Purpose:** Dump profiling metrics for a task to log output
- **Inputs:** `inTask` ΓÇö pointer to task
- **Outputs/Return:** None
- **Side effects:** Logs task metadata and profiling data using `logDump*` macros
- **Calls:** Logging macros (with helper macros `DUMPIT_ZU` / `DUMPIT_ZS`)
- **Notes:** Only compiled in DEBUG builds; uses preprocessor stringification to label fields

## Control Flow Notes
**Initialization:** `mytm_initialize()` must be called once before any other `mytm_*` calls to create the global mutex.

**Task Lifecycle:**
1. `myTMSetup()` / `myXTMSetup()` create a task and spawn its thread
2. `thread_loop()` runs in background, calling the callback periodically with drift correction
3. Task can be reset via `myTMReset()` (resync to current time) or removed via `myTMRemove()` (signal stop)
4. `myTMCleanup()` is called periodically to collect finished tasks and free memory

**Synchronization:** Only one task executes its callback at a time (protected by `sTMTaskMutex`). The main thread is not locked out by TMTasks, but threading priority ensures tasks preempt the main thread if they don't block.

## External Dependencies
- **SDL_thread.h:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`
- **SDL_timer.h:** `SDL_GetTicks()`, `SDL_Delay()`
- **SDL_error.h:** `SDL_GetError()`
- **thread_priority_sdl.h:** `BoostThreadPriority()`
- **Logging.h:** `logWarning()`, `logAnomaly*()`, `logDump*()` (variadic logging macros)
- **cseries.h, mytm.h:** Header declarations and macros; `obj_clear()` (trivial helper, likely defined in cseries)
- **std::vector:** Contained in `<vector>` via `using std::vector` (guarded by `!NO_STD_NAMESPACE`)
