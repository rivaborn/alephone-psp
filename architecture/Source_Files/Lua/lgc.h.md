# Source_Files/Lua/lgc.h

## File Purpose
Garbage collector interface header for Lua's incremental tri-color marking GC. Defines GC state constants, bit manipulation utilities for object marking, tri-color classification macros, write barriers to preserve GC invariants, and function declarations for GC operations.

## Core Responsibilities
- Define GC finite-state machine states (pause, propagate, sweep string, sweep, finalize)
- Provide bit-level utilities for manipulating and testing the `marked` field of collectable objects
- Implement tri-color marking scheme: two white colors (for generational collection), black, and implicit gray
- Supply write barrier macros (`luaC_barrier`, `luaC_barriert`, etc.) to detect and handle blackΓåÆwhite references that violate GC invariants
- Declare entry-point functions for step-wise GC execution, full GC cycles, object linking, and finalization
- Expose thread-safe GC threshold checks for incremental collection triggering

## Key Types / Data Structures
None. (File defines only macro constants and function declarations; uses types from `lobject.h`.)

## Global / File-Static State
None.

## Key Functions / Methods

### luaC_step
- Signature: `void luaC_step(lua_State *L)`
- Purpose: Perform one incremental step of the garbage collector state machine
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: None
- Side effects: Modifies GC state, marks/sweeps objects, may deallocate memory
- Calls: (Not visible in this file; implementation in lgc.c)
- Notes: Called by `luaC_checkGC` macro when GC threshold is exceeded; driven by allocation

### luaC_fullgc
- Signature: `void luaC_fullgc(lua_State *L)`
- Purpose: Execute a complete garbage collection cycle in one call
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: None
- Side effects: Marks and sweeps entire heap; may deallocate many objects
- Calls: (Not visible in this file)
- Notes: Blocking call; runs GC to completion rather than incrementally

### luaC_link
- Signature: `void luaC_link(lua_State *L, GCObject *o, lu_byte tt)`
- Purpose: Link a newly created GCObject into the GC linked list
- Inputs: `L` ΓÇö Lua state; `o` ΓÇö new object; `tt` ΓÇö object type tag
- Outputs/Return: None
- Side effects: Inserts `o` at head of GC chain; marks object with appropriate color based on current GC state
- Calls: (Not visible in this file)
- Notes: Must be called immediately after allocating a collectable object

### luaC_barrierf
- Signature: `void luaC_barrierf(lua_State *L, GCObject *o, GCObject *v)`
- Purpose: Forward write barrier; called when a black object `o` is assigned a reference to white object `v`
- Inputs: `L` ΓÇö Lua state; `o` ΓÇö black object being modified; `v` ΓÇö white object being referenced
- Outputs/Return: None
- Side effects: Marks `v` gray to preserve the "no blackΓåÆwhite pointers" invariant during incremental marking
- Calls: (Not visible in this file)
- Notes: Used by `luaC_barrier` and `luaC_objbarrier` macros; essential for tri-color correctness

### luaC_barrierback
- Signature: `void luaC_barrierback(lua_State *L, Table *t)`
- Purpose: Backward/table write barrier; called when a table's value is modified
- Inputs: `L` ΓÇö Lua state; `t` ΓÇö table being modified
- Outputs/Return: None
- Side effects: Marks table for re-scanning if it is black
- Calls: (Not visible in this file)
- Notes: Less expensive than forward barrier; used by `luaC_barriert` and `luaC_objbarriert` macros

### luaC_freeall
- Signature: `void luaC_freeall(lua_State *L)`
- Purpose: Free all GC-managed objects during VM shutdown
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: None
- Side effects: Deallocates entire heap; calls finalizers
- Calls: (Not visible in this file)
- Notes: Called at VM teardown; bypasses normal GC logic

### luaC_callGCTM, luaC_separateudata, luaC_linkupval
- Declarations only; implementation not visible in this file. Used for finalization and upvalue management.

## Control Flow Notes

**GC State Machine:** The GC transitions through states defined by `GCSpause`, `GCSpropagate`, `GCSsweepstring`, `GCSsweep`, `GCSfinalize`. `luaC_step` advances the state; `luaC_fullgc` cycles through all states to completion.

**Incremental Execution:** `luaC_checkGC` macro (invoked by allocators) checks if `totalbytes >= GCthreshold` and calls `luaC_step` to perform one GC step without blocking.

**Tri-Color Invariant:** Objects are marked as white (two variants for generational sweeping) or black; unmarked objects are implicitly gray. The critical invariantΓÇöno black object may reference a white objectΓÇöis enforced via write barriers. When a write would violate this, the barrier function grays the target to restore the invariant.

**Object Lifecycle:** New objects are linked via `luaC_link` (colored based on current GC phase), traced during propagation, and swept in `GCSsweep` or `GCSsweepstring`. Finalizers run in `GCSfinalize`.

## External Dependencies
- `#include "lobject.h"` ΓÇö Provides `GCObject`, `GCheader`, `Table`, `UpVal`, `lua_State` (via transitive includes)
- Macro helpers: `cast()`, `check_exp()` (from llimits.h via lobject.h)
- Type aliases: `lu_byte`, `size_t` (from llimits.h or standard headers)
