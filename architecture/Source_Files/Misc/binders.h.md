# Source_Files/Misc/binders.h

## File Purpose
Provides a template-based synchronization mechanism for two-way data binding between paired objects. Implements a generic pattern where two `Bindable` objects can export/import state to keep themselves synchronized, managed collectively via a `BinderSet` container.

## Core Responsibilities
- Define the `Bindable<T>` interface for types that can export and import values
- Provide the abstract `ABinder` base for binding operations
- Implement `Binder<T>` to synchronize state between two `Bindable` objects in both directions
- Maintain `BinderSet` to manage multiple binders and execute batch synchronization operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Bindable<T>` | Template abstract class | Interface for objects that can export/import data of type `T` |
| `ABinder` | Abstract class | Polymorphic base for all binders (enables type-erased container) |
| `Binder<T>` | Template class | Concrete binder linking two `Bindable<T>` objects for bidirectional sync |
| `BinderSet` | Class | Container managing multiple binders; executes batch sync operations |

## Global / File-Static State
None.

## Key Functions / Methods

### Bindable<T>::bind_export()
- Signature: `virtual T bind_export () = 0`
- Purpose: Export the current state as a value of type `T`
- Inputs: None
- Outputs/Return: Value of type `T` representing current state
- Side effects: None (pure virtual)
- Notes: Pure virtual; must be implemented by derived classes

### Bindable<T>::bind_import(T)
- Signature: `virtual void bind_import (T) = 0`
- Purpose: Import and apply a state value of type `T`
- Inputs: `T` ΓÇö the state value to import
- Outputs/Return: None
- Side effects: Modifies internal state of the object
- Notes: Pure virtual; must be implemented by derived classes

### Binder<T>::migrate_first_to_second()
- Signature: `void migrate_first_to_second ()`
- Purpose: Synchronize state from `thing1` to `thing2`
- Inputs: None (reads from `thing1`)
- Outputs/Return: None
- Side effects: Exports from `thing1`, imports into `thing2`
- Calls: `thing1->bind_export()`, `thing2->bind_import()`

### Binder<T>::migrate_second_to_first()
- Signature: `void migrate_second_to_first ()`
- Purpose: Synchronize state from `thing2` to `thing1`
- Inputs: None (reads from `thing2`)
- Outputs/Return: None
- Side effects: Exports from `thing2`, imports into `thing1`
- Calls: `thing2->bind_export()`, `thing1->bind_import()`

### BinderSet::insert()
- Signature: `template<typename T> void insert (Bindable<T>* first, Bindable<T>* second)`
- Purpose: Register a new binder pair for two `Bindable` objects
- Inputs: Two pointers to `Bindable<T>` objects
- Outputs/Return: None
- Side effects: Allocates a new `Binder<T>` on the heap, adds to `m_list`
- Calls: `Binder<T>` constructor
- Notes: Null-pointer safe; only inserts if both pointers are valid

### BinderSet::migrate_all_first_to_second()
- Signature: `void migrate_all_first_to_second ()`
- Purpose: Synchronize all registered binders from firstΓåÆsecond
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `migrate_first_to_second()` on every binder in the set
- Calls: `std::for_each` with `call_first_second` functor
- Notes: Batch operation; useful for coordinated state sync

### BinderSet::migrate_all_second_to_first()
- Signature: `void migrate_all_second_to_first ()`
- Purpose: Synchronize all registered binders from secondΓåÆfirst
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `migrate_second_to_first()` on every binder in the set
- Calls: `std::for_each` with `call_second_first` functor
- Notes: Batch operation; inverse of `migrate_all_first_to_second()`

## Control Flow Notes
Not inferable from this file. Likely used during state sync operations (e.g., configuration save/load, level bind/unbind, or runtime state reconciliation). The pattern suggests a two-phase commit: accumulate binder pairs during setup, then trigger batch synchronization in one direction or the other.

## External Dependencies
- `<list>`: `std::list` for dynamic binder storage
- `<algorithm>`: `std::for_each` for batch operations
