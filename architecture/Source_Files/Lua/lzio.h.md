# Source_Files/Lua/lzio.h

## File Purpose

Header for Lua's buffered stream (ZIO) I/O system. Provides macros and function declarations for reading data from custom sources (files, memory, network, etc.) via a reader callback pattern, with support for dynamic buffering.

## Core Responsibilities

- Define the `Zio` buffered stream struct for reading from custom sources
- Define the `Mbuffer` dynamic memory buffer struct
- Provide macros for buffer access, resizing, and character extraction
- Declare initialization and reading functions
- Support the reader callback pattern for flexible input sources

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Zio` | struct | Buffered input stream: tracks unread bytes, current position, reader callback, and associated Lua state |
| `Mbuffer` | struct | Dynamic memory buffer: holds char pointer, current size, and allocated capacity |

## Global / File-Static State

None.

## Key Functions / Methods

### luaZ_init
- Signature: `void luaZ_init(lua_State *L, ZIO *z, lua_Reader reader, void *data)`
- Purpose: Initialize a ZIO stream object with a reader callback and context data
- Inputs: `L` = Lua state, `z` = ZIO struct to initialize, `reader` = callback function, `data` = opaque context passed to reader
- Outputs/Return: None (void)
- Side effects: Modifies ZIO struct fields
- Calls: (implementation not shown)
- Notes: Must be called before using ZIO with `zgetc` or `luaZ_read`

### luaZ_read
- Signature: `size_t luaZ_read(ZIO* z, void* b, size_t n)`
- Purpose: Read next `n` bytes from the stream into buffer `b`
- Inputs: `z` = ZIO stream, `b` = destination buffer, `n` = number of bytes
- Outputs/Return: Number of bytes actually read
- Side effects: Advances stream position; may call reader callback
- Calls: Likely calls `luaZ_fill` internally
- Notes: Returns fewer bytes than requested at end-of-stream

### luaZ_lookahead
- Signature: `int luaZ_lookahead(ZIO *z)`
- Purpose: Peek at the next character without consuming it
- Inputs: `z` = ZIO stream
- Outputs/Return: Next character as int, or `EOZ` (-1) at end-of-stream
- Side effects: May refill buffer
- Calls: (implementation not shown)

### luaZ_fill
- Signature: `int luaZ_fill(ZIO *z)`
- Purpose: Refill the internal buffer by calling the reader callback (private, internal use)
- Inputs: `z` = ZIO stream
- Outputs/Return: Character from refilled buffer or `EOZ` at stream end
- Side effects: Calls reader callback; updates ZIO buffer pointers
- Calls: Calls the `lua_Reader` callback stored in ZIO
- Notes: Called by `zgetc` macro when buffer is exhausted

### luaZ_openspace
- Signature: `char *luaZ_openspace(lua_State *L, Mbuffer *buff, size_t n)`
- Purpose: Ensure the buffer has space for at least `n` bytes
- Inputs: `L` = Lua state, `buff` = Mbuffer to resize, `n` = required size
- Outputs/Return: Pointer to usable buffer space
- Side effects: May reallocate and resize buffer
- Calls: (implementation not shown)
- Notes: Simplifies safe writing to dynamic buffers

## Macros

- **`zgetc(z)`**: Extract next character from stream; returns `char2int(*(z)->p++)` if bytes remain, else calls `luaZ_fill(z)`
- **`char2int(c)`**: Cast unsigned char to int safely
- **`luaZ_initbuffer`, `luaZ_resetbuffer`**: Buffer initialization/reset
- **`luaZ_resizebuffer`**: Reallocate buffer to new size
- **`luaZ_buffer`, `luaZ_sizebuffer`, `luaZ_bufflen`**: Buffer accessor macros

## Control Flow Notes

This module is part of Lua's **parse/load infrastructure**. When the Lua interpreter loads code (via `lua_load`), it uses a ZIO stream to read bytes from the source through a reader callback, allowing transparent support for files, strings, network streams, etc. The buffering layer optimizes performance by batching reader calls.

## External Dependencies

- `lua.h`: Defines `lua_State`, `lua_Reader` callback typedef
- `lmem.h`: Memory allocation/reallocation macros (`luaM_reallocvector`, `luaM_toobig`)
- `config.h`: Conditional compilation (e.g., `HAVE_LUA`)
