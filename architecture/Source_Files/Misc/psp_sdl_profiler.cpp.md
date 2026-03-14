# Source_Files/Misc/psp_sdl_profiler.cpp

## File Purpose
Implementation of a hierarchical performance profiler for PSP/SDL-based games. Tracks function execution time, call counts, and averages while maintaining parent-child relationships for nested profiling.

## Core Responsibilities
- Initialize and manage a profiler instance for a single function
- Bracket function execution with `begin()` / `end()` to measure elapsed time
- Accumulate total execution time and average time across calls
- Build and maintain a hierarchical tree of profiled functions (parent/child)
- Export profiling statistics as an XML-formatted text dump

## Key Types / Data Structures
None (class definition in header).

## Global / File-Static State
None.

## Key Functions / Methods

### PSPSDLProfiler (constructor)
- Signature: `PSPSDLProfiler(const char* functionName)`
- Purpose: Initialize a new profiler instance for a named function.
- Inputs: `functionName` (C-string naming the profiled function)
- Outputs/Return: None
- Side effects: Initializes `avarageTime = 0.0f`, `totalCalls = 0`, `lastTicks = 0`; calls `setFunctionName()`.
- Calls: `setFunctionName()`
- Notes: Does not initialize `totalTime` memberΓÇöuninitialized on construction.

### begin
- Signature: `void begin(void)`
- Purpose: Mark the start of a profiled code section; record current SDL tick count.
- Inputs: None
- Outputs/Return: None
- Side effects: Increments `totalCalls`, updates `lastTicks` via `SDL_GetTicks()`.
- Calls: `SDL_GetTicks()`

### end
- Signature: `void end(void)`
- Purpose: Mark the end of profiled section; accumulate elapsed time into totals and running average.
- Inputs: None
- Outputs/Return: None
- Side effects: Computes delta time, updates `totalTime` and `avarageTime`.
- Calls: `SDL_GetTicks()`
- Notes: **Averaging bug**: divides `avarageTime` by 2.0 when `totalCalls > 1`, which is not a correct running average formula; should be `avarageTime = totalTime / totalCalls`.

### dump
- Signature: `void dump(FILE* f, unsigned int level = 0)`
- Purpose: Recursively output the profiler tree as indented XML to file.
- Inputs: `f` (FILE* for output), `level` (recursion depth for indentation).
- Outputs/Return: None
- Side effects: Writes formatted XML output to file `f`; calls `dump()` on all children.
- Calls: `fprintf()`, `getChild(i)->dump()`, accessor methods.
- Notes: Outputs: function name, parent name, call count, average time, total time; null-check on `f` prevents crashes.

### setFunctionName, setParent, addChild
- Purpose: Configure function name, parent profiler, and child profilers respectively.
- `addChild` prevents duplicate children via linear search before insertion.

## Control Flow Notes
Intended usage pattern: construct a `PSPSDLProfiler`, call `begin()` at function entry, `end()` at exit, optionally building a parent-child tree, and finally `dump()` to generate a profiling report. Fits a frame/update profiling model where timing is collected across frames.

## External Dependencies
- `<cstdio>` ΓÇö `fprintf()`, `FILE*`
- `<cstring>` ΓÇö `strncpy()`
- `<SDL.h>` ΓÇö `SDL_GetTicks()` (millisecond tick counter)
- Included header: `psp_sdl_profiler.h` (class definition, vector container)
