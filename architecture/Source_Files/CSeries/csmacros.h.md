# Source_Files/CSeries/csmacros.h

## File Purpose
Header file providing utility macros and template functions for the Aleph One game engine. Supplies common mathematical operations, bit manipulation helpers, memory management wrappers, and bounds-checked array accessΓÇödesigned to reduce boilerplate and improve type safety across the codebase.

## Core Responsibilities
- Math utilities: min, max, floor, ceiling, clamping, absolute value, sign extraction
- Bit manipulation: flag operations on 32-bit and 16-bit integers
- Generic swap template for any type
- Rectangle dimension calculation helpers
- Power-of-two computation
- Bounds-checked array access with bounds validation
- Type-safe wrappers around `memcpy()` and `memset()` to eliminate explicit `sizeof()` calls

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### SWAP
- Signature: `template <typename T> void SWAP(T& a, T& b)`
- Purpose: Swap two values of the same type without temporary variable overhead (compiler may optimize).
- Inputs: Two references of type `T`.
- Outputs/Return: None (void).
- Side effects: Modifies both input references in-place.
- Calls: None visible.
- Notes: Simple but safe; relies on compiler optimization for efficiency.

### NextPowerOfTwo
- Signature: `static inline int NextPowerOfTwo(int n)`
- Purpose: Return the smallest power of two greater than or equal to `n`.
- Inputs: Integer `n`.
- Outputs/Return: Power of two ΓëÑ `n`.
- Side effects: None.
- Calls: None.
- Notes: Bit-shifting loop; no bounds checking (overflow risk if `n` is very large).

### GetMemberWithBounds
- Signature: `template<class T> T* GetMemberWithBounds(T* Array, const size_t Index, const size_t Number)`
- Purpose: Access array element at `Index` with bounds checking.
- Inputs: Pointer to array, index, and number of valid elements.
- Outputs/Return: Pointer to element if in bounds; `NULL` otherwise.
- Side effects: None.
- Calls: None.
- Notes: Caller must check for `NULL` return; only validates `0 <= Index < Number`.

### obj_copy / objlist_copy
- Signature: `template<class T> void obj_copy(T& destination, const T& source)` and `objlist_copy(T* destination, const T* source, size_t num_objects)`
- Purpose: Type-safe wrapper around `memcpy()` for single objects or arrays.
- Inputs: Source and destination references/pointers; for lists, count of objects.
- Outputs/Return: None (void).
- Side effects: Modifies destination memory.
- Calls: `memcpy()`.

### obj_set / objlist_set / obj_clear / objlist_clear
- Purpose: Type-safe `memset()` wrappers (fill with value or zero).
- Signature: `obj_set(T& object, int value)`, `objlist_set(T* object_list, int value, size_t num_objects)`, `obj_clear(T& object)`, `objlist_clear(T* object_list, size_t num_objects)`.
- Inputs: Object(s) reference/pointer; for `*_set`, fill value; for lists, count.
- Outputs/Return: None (void).
- Side effects: Overwrites object memory.
- Calls: `memset()` (or `obj_set()` for clear).

## Control Flow Notes
This is a header file; no control flow. Macros are expanded inline at compile time. Template functions are instantiated per-type at call sites. Conditional compilation with `#ifdef __cplusplus` ensures C++ templates are only compiled in C++ contexts.

## External Dependencies
- `<string.h>`: `memcpy()`, `memset()`
