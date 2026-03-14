# Source_Files/Misc/binders.h

## File Purpose
Defines a bidirectional synchronization system for maintaining consistent state between paired objects. The `Bindable` interface and `Binder` template allow two objects to export/import state, while `BinderSet` orchestrates bulk synchronization across multiple binder pairs.

## Core Responsibilities
- Define the `Bindable<T>` interface for state export/import operations
- Implement `Binder<T>` to pair and synchronize two `Bindable` objects in both directions
- Provide `BinderSet` to manage and batch-execute migration across all registered binders
- Support polymorphic binder operations through the `ABinder` base class

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Bindable<T>` | Template class (interface) | Abstract contract for objects that can export/import state of type T |
| `ABinder` | Abstract class | Non-template base for binder polymorphism and container storage |
| `Binder<T>` | Template class (concrete) | Synchronizes state between two `Bindable<T>` objects in both directions |
| `BinderSet` | Class (container) | Manages a list of heterogeneous `ABinder` pointers and executes bulk migrations |

## Global / File-Static State
None.

## Key Functions / Methods

### Bindable<T>::bind_export
- Signature: `virtual T bind_export() = 0`
- Purpose: Extract/serialize state from this object
- Outputs/Return: State of type T ready for import into another object
- Notes: Pure virtual; implementation-defined for each bindable type

### Bindable<T>::bind_import
- Signature: `virtual void bind_import(T) = 0`
- Purpose: Receive and apply state from a paired object
- Inputs: State of type T
- Notes: Pure virtual; implementation-defined for each bindable type

### Binder<T>::migrate_first_to_second
- Signature: `void migrate_first_to_second()`
- Purpose: Sync state from `thing1` ΓåÆ `thing2`
- Side effects: Calls `thing1->bind_export()` and `thing2->bind_import()`
- Calls: `bind_export()`, `bind_import()`

### Binder<T>::migrate_second_to_first
- Signature: `void migrate_second_to_first()`
- Purpose: Sync state from `thing2` ΓåÆ `thing1`
- Side effects: Calls `thing2->bind_export()` and `thing1->bind_import()`
- Calls: `bind_export()`, `bind_import()`

### BinderSet::insert
- Signature: `template<typename T> void insert(Bindable<T>* first, Bindable<T>* second)`
- Purpose: Register a new binder pair
- Inputs: Two non-null `Bindable<T>` pointers
- Side effects: Allocates and stores new `Binder<T>` if both pointers are valid
- Notes: Silently ignores if either pointer is null

### BinderSet::migrate_all_first_to_second
- Signature: `void migrate_all_first_to_second()`
- Purpose: Synchronize all registered binders in one direction
- Side effects: Calls `migrate_first_to_second()` on each binder
- Calls: `for_each(call_first_second)`

### BinderSet::migrate_all_second_to_first
- Signature: `void migrate_all_second_to_first()`
- Purpose: Synchronize all registered binders in opposite direction
- Side effects: Calls `migrate_second_to_first()` on each binder
- Calls: `for_each(call_second_first)`

## Control Flow Notes
Not inferable from this file. This is a utility library likely invoked during state synchronization phases (e.g., after input processing, before rendering, or during network replication). The bidirectional migrate functions suggest use cases where state must be kept in sync between two representations (e.g., authoritative state and local cache, or game logic and network state).

## External Dependencies
- `<list>` ΓÇô std::list for heterogeneous binder storage
- `<algorithm>` ΓÇô std::for_each for iteration over binder list
