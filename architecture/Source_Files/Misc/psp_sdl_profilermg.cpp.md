# Source_Files/Misc/psp_sdl_profilermg.cpp

## File Purpose
Manager class that maintains a collection of `PSPSDLProfiler` instances and orchestrates profiling call hierarchies. Coordinates begin/end profiling events, tracks parent-child relationships between profiled functions, and exports timing data to XML.

## Core Responsibilities
- Create profiler instances on-demand and cache them by function name
- Establish and maintain parent-child call hierarchies via `lastCalled` stack
- Route begin/end profiling calls to the appropriate `PSPSDLProfiler` object
- Clean up allocated profiler objects on destruction
- Serialize profiling results to XML files

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `profilers` | `std::vector<PSPSDLProfiler*>` | Dynamic array storing all managed profiler instances |
| `lastCalled` | `PSPSDLProfiler*` | Pointer to the currently active (innermost) profiler in the call stack |

## Global / File-Static State
None.

## Key Functions / Methods

### PSPSDLProfilerMg (constructor)
- **Signature:** `PSPSDLProfilerMg()`
- **Purpose:** Initialize the profiler manager
- **Inputs:** None
- **Outputs/Return:** (constructor)
- **Side effects:** Initializes `lastCalled` to `NULL`
- **Calls:** None
- **Notes:** No dynamic allocation; vector is empty at construction

### ~PSPSDLProfilerMg (destructor)
- **Signature:** `~PSPSDLProfilerMg()`
- **Purpose:** Clean up all managed profiler instances
- **Inputs:** None
- **Outputs/Return:** (destructor)
- **Side effects:** Deletes each non-null `PSPSDLProfiler*` in `profilers` vector
- **Calls:** `delete` on each profiler
- **Notes:** Assumes all pointers are valid; no null-safety issues due to check

### begin
- **Signature:** `void begin(PSPSDLProfiler* p)`
- **Purpose:** Start profiling a function and establish its position in the call hierarchy
- **Inputs:** `p` ΓÇô pointer to profiler to activate (null-check performed)
- **Outputs/Return:** None
- **Side effects:** Sets parent relationship, updates `lastCalled`, calls `p->begin()`
- **Calls:** `p->setParent(lastCalled)`, `lastCalled->addChild(p)`, `p->begin()`
- **Notes:** If `lastCalled` exists, `p` is added as its child before becoming the new `lastCalled`; creates a dynamic call tree

### end (overload: PSPSDLProfiler*)
- **Signature:** `void end(PSPSDLProfiler* p)`
- **Purpose:** Stop profiling for a given profiler and restore the parent as active
- **Inputs:** `p` ΓÇô pointer to profiler to deactivate (null-check performed)
- **Outputs/Return:** None
- **Side effects:** Calls `p->end()`, restores `lastCalled` to the parent profiler
- **Calls:** `p->end()`, `p->getParent()`
- **Notes:** Effectively pops the profiler from the active call stack

### end (overload: const char*)
- **Signature:** `void end(const char* functionName)`
- **Purpose:** Stop profiling by function name (convenience wrapper)
- **Inputs:** `functionName` ΓÇô C-string name to look up
- **Outputs/Return:** None
- **Side effects:** Calls the `PSPSDLProfiler*` overload after lookup
- **Calls:** `getProfiler(functionName)`, `end(PSPSDLProfiler*)`
- **Notes:** Decouples caller from needing to hold profiler pointers

### createProfiler
- **Signature:** `PSPSDLProfiler* createProfiler(const char* functionName, bool begin = true)`
- **Purpose:** Create a new profiler for a function, or retrieve an existing one; optionally start it
- **Inputs:** `functionName` ΓÇô name of function to profile; `begin` ΓÇô whether to immediately call `begin()` (default true)
- **Outputs/Return:** Pointer to created/retrieved profiler, or `NULL` if allocation fails
- **Side effects:** Allocates new `PSPSDLProfiler` and adds to `profilers` vector if not already present; may call `begin()` internally
- **Calls:** `getProfiler(functionName)`, `new PSPSDLProfiler(functionName)`, `profilers.push_back()`, `this->begin(p)`
- **Notes:** Idempotent for the lookup; memoizes profilers by function name

### getProfiler
- **Signature:** `PSPSDLProfiler* PSPSDLProfilerMg::getProfiler(const char* functionName)`
- **Purpose:** Retrieve a profiler instance by function name
- **Inputs:** `functionName` ΓÇô C-string name to search for
- **Outputs/Return:** Pointer to matching profiler, or `NULL` if not found
- **Side effects:** None
- **Calls:** `strcmp()`, `p->getFunctionName()`
- **Notes:** Linear O(n) search; skips null entries; case-sensitive string comparison

### dump
- **Signature:** `void dump(const char* fileName)`
- **Purpose:** Export profiling data to an XML file
- **Inputs:** `fileName` ΓÇô path/name of output file
- **Outputs/Return:** None
- **Side effects:** Creates/overwrites file, writes XML header and profile tree; may fail silently if file open fails
- **Calls:** `fopen()`, `fprintf()`, `profilers[0]->dump(FILE*, unsigned int)`, `fclose()`
- **Notes:** Uses first profiler as root of hierarchy; empty vector or file I/O failure causes early return with no error reporting

## Control Flow Notes
This manager operates independently of typical frame/update/render cycles. It is invoked via compile-time macros (`PSPROF_BEGIN`, `PSPROF_END`) that expand only on PSP platform. The call hierarchy is built dynamically as functions nest: each `begin()` pushes a profiler onto the implicit stack (via `lastCalled`), and each `end()` pops it. The XML dump reconstructs this hierarchy for offline analysis.

## External Dependencies
- **Standard library:** `<cstdlib>`, `<cstdio>` (for `FILE`, `fopen`, `fprintf`, `fclose`), `<cstring>` (for `strcmp`)
- **Engine headers:** `psp_sdl_profilermg.h` (class declaration), `psp_sdl_profiler.h` (profiler interface)
- **STL:** `std::vector` (via header)
- **External symbols used but not defined here:** `PSPSDLProfiler` class and all its methods (defined in `psp_sdl_profiler.h/cpp`)
