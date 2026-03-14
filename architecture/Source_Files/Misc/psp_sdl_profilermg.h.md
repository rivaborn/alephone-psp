# Source_Files/Misc/psp_sdl_profilermg.h

## File Purpose
Manager class for PSP SDL profilers that maintains a collection of profiler instances and provides operations to create, retrieve, and manage performance profiling. Includes conditional compilation macros for profiling instrumentation on PSP platform builds.

## Core Responsibilities
- Create and manage a collection of `PSPSDLProfiler` instances indexed by function name
- Provide simplified API for profiling begin/end operations via macros
- Track profiler access patterns (most recently called profiler)
- Export accumulated profiling data to file for analysis

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PSPSDLProfilerMg | class | Manager container for multiple profilers with lifecycle and data export |
| PSPROF_MARK_BEGIN / PSPROF_MARK_END / PSPROF_BEGIN / PSPROF_END | macros | Conditional profiling instrumentation; no-op on non-PSP platforms |

## Global / File-Static State
None.

## Key Functions / Methods

### createProfiler
- Signature: `PSPSDLProfiler* createProfiler(const char* functionName, bool begin = true)`
- Purpose: Create or retrieve a profiler for the named function; optionally begin timing immediately
- Inputs: Function name string, begin flag (default true)
- Outputs/Return: Pointer to created/managed profiler
- Side effects: Allocates profiler if not present, stores in vector, may start timing
- Calls: PSPSDLProfiler constructor (inferred), `begin()`
- Notes: Returns managed profiler; lifecycle tied to manager

### getProfiler
- Signature: `PSPSDLProfiler* getProfiler(const char* functionName)`
- Purpose: Retrieve existing profiler by function name
- Inputs: Function name string
- Outputs/Return: Pointer to profiler or null if not found
- Side effects: None
- Calls: None visible
- Notes: Read-only lookup

### begin / end
- Signature: `void begin(PSPSDLProfiler* p)` / `void end(PSPSDLProfiler* p)` / `void end(const char* functionName)`
- Purpose: Start/stop timing on a profiler; overloaded end accepts profiler pointer or function name
- Inputs: Profiler pointer or function name
- Outputs/Return: None
- Side effects: Updates `lastCalled`; delegates to profiler's timing methods
- Calls: PSPSDLProfiler::begin(), PSPSDLProfiler::end(), getProfiler()
- Notes: end(const char*) variant must resolve name to profiler

### dump
- Signature: `void dump(const char* fileName)`
- Purpose: Export profiling statistics to file
- Inputs: Output file path
- Outputs/Return: None
- Side effects: I/O; writes all profiler data
- Calls: PSPSDLProfiler::dump(), file operations
- Notes: Iterates all managed profilers

## Control Flow Notes
Profiling is typically instrumented around function bodies via `PSPROF_BEGIN(manager)` / `PSPROF_END(manager)` macros. On PSP builds, these expand to `createProfiler(__FUNCTION__)` and `end(__FUNCTION__)` calls; on other platforms they are no-ops. Manager accumulates timing data across frames and can dump results at shutdown or on-demand.

## External Dependencies
- `<vector>` (STL container for profiler collection)
- `psp_sdl_profiler.h` (PSPSDLProfiler class definition; provides timing, hierarchical profiling, child/parent relationships)
