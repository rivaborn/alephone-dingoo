# Expat/xmlparse/hashtable.h

## File Purpose
Header file defining the hash table API and data structures for the Expat XML parser. Provides lookup, initialization, destruction, and iteration operations over a simple hash table with unicode/ASCII key support.

## Core Responsibilities
- Define hash table data structure and key type (conditional on unicode configuration)
- Declare hash table lifecycle operations (init, destroy)
- Declare lookup/insert operation
- Provide iterator interface for traversing entries

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `KEY` | typedef | String key type; points to `wchar_t` or `unsigned short` (Unicode) or `char` (ASCII) depending on `XML_UNICODE*` macros |
| `NAMED` | struct | Single hash table entry containing a key (`name` field) |
| `HASH_TABLE` | struct | Main hash table: `v` (bucket array), `size` (capacity), `used` (entries), `usedLim` (resize threshold) |
| `HASH_TABLE_ITER` | struct | Iterator state: `p` (current bucket), `end` (boundary) |

## Global / File-Static State
None.

## Key Functions / Methods

### lookup
- Signature: `NAMED *lookup(HASH_TABLE *table, KEY name, size_t createSize)`
- Purpose: Search for entry by key; create with given size if not found
- Inputs: hash table, key name, allocation size for new entry
- Outputs/Return: pointer to `NAMED` entry (existing or newly allocated)
- Side effects: modifies table (allocation, bucket reorganization on resize)
- Calls: Not visible in header; likely malloc/realloc internally
- Notes: `createSize == 0` likely means "lookup-only" (no insertion)

### hashTableInit
- Signature: `void hashTableInit(HASH_TABLE *)`
- Purpose: Initialize empty hash table
- Inputs: uninitialized hash table struct
- Outputs/Return: none
- Side effects: zeroes/initializes table fields

### hashTableDestroy
- Signature: `void hashTableDestroy(HASH_TABLE *)`
- Purpose: Deallocate hash table and all entries
- Inputs: initialized hash table
- Outputs/Return: none
- Side effects: frees bucket array and entries

### hashTableIterInit
- Signature: `void hashTableIterInit(HASH_TABLE_ITER *, const HASH_TABLE *)`
- Purpose: Reset iterator to table start
- Inputs: iterator struct, table
- Outputs/Return: none
- Side effects: sets iterator position to first bucket

### hashTableIterNext
- Signature: `NAMED *hashTableIterNext(HASH_TABLE_ITER *)`
- Purpose: Advance iterator and return next entry
- Inputs: iterator
- Outputs/Return: pointer to next `NAMED`, or NULL when exhausted
- Side effects: updates iterator position

## Control Flow Notes
Not inferable from header; typical usage pattern: `hashTableInit` ΓåÆ `lookup` (repeated) ΓåÆ iterate with `hashTableIterInit`/`hashTableIterNext` ΓåÆ `hashTableDestroy`.

## External Dependencies
- `<stddef.h>` ΓÇö for `size_t`
- Conditional compilation symbols: `XML_UNICODE`, `XML_UNICODE_WCHAR_T` (defined elsewhere)
