# Source_Files/Misc/psp_sdl_profiler.h

## File Purpose
Header for a hierarchical profiler class designed to measure function/code-section execution time on PSP (PlayStation Portable) platforms. Tracks timing statistics per profiled function and maintains a parent-child tree to correlate nested calls.

## Core Responsibilities
- Track execution time (average, total, call count) for profiled functions
- Manage a hierarchical tree of profilers (parent-child relationships)
- Provide timing queries for performance analysis
- Serialize profiling results to file output for inspection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PSPSDLProfiler | class | Profiler for a single function or code section; maintains timing stats and hierarchy |
| childs | vector<PSPSDLProfiler*> | Collection of child profilers (nested calls) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MAX_FUNC_NAME | static const int | file-static | Max length for function name buffer (1024 chars) |

## Key Functions / Methods

### Constructor
- Signature: `PSPSDLProfiler(const char* functionName)`
- Purpose: Initialize a profiler instance for a named function
- Inputs: Function name (C-string)
- Outputs/Return: None (constructor)
- Side effects: Allocates and initializes member state
- Notes: Function name copied into fixed-size buffer (potential overflow if >1024 chars)

### begin / end
- Signature: `void begin()`, `void end()`
- Purpose: Mark the start and end of a profiled execution window
- Outputs/Return: None (update internal state)
- Side effects: Likely capture timestamps; `end()` updates totalTime, totalCalls, avarageTime
- Calls: Likely platform-specific timing calls (not visible in header)
- Notes: Timing logic not shown; `lastTicks` suggests low-level tick capture

### dump
- Signature: `void dump(FILE* f, unsigned int level = 0)`
- Purpose: Recursively output profiling statistics to file in hierarchical format
- Inputs: File pointer, hierarchy level (for indentation)
- Outputs/Return: None (I/O only)
- Side effects: Writes to file
- Calls: Presumably recursive on children
- Notes: Level parameter enables formatted tree output

### Hierarchy Management
- `setParent()` / `getParent()`: Assign/retrieve parent profiler
- `addChild()`: Register a child profiler (nested call)
- `getChild(unsigned int num)`: Fetch child by index (bounds-checked; returns NULL if out-of-range)
- `getChildsCount()`: Return number of children

## Control Flow Notes
This profiler is designed for runtime instrumentation during a game loop. Typical usage:
1. Create a `PSPSDLProfiler` for a function at definition or scope entry
2. Call `begin()` at code entry; `end()` at exit (likely wrapped in constructor/destructor)
3. Parent-child links are established to correlate nested calls
4. At shutdown or debug time, call `dump()` to output the hierarchy to a file for analysis

Not inferable: whether begin/end use stack-allocated RAII or explicit calls; exact timing mechanism (clock ticks, microseconds, etc.).

## External Dependencies
- `<cstdio>`: FILE for dump output
- `<vector>`: std::vector for child storage
- Timing API (not in header): Platform-specific clock/tick functions called by begin/end
