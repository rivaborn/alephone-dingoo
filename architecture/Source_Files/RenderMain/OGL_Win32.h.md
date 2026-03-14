# Source_Files/RenderMain/OGL_Win32.h

## File Purpose
Windows-specific OpenGL header that manages ARB extension function pointers for the rendering system. Handles runtime loading of optional OpenGL features like multitexturing and texture compression on Windows.

## Core Responsibilities
- Define function pointer types for ARB (OpenGL Architecture Review Board) extensions
- Conditionally define or declare extension function pointers based on compilation context
- Provide convenience macros that abstract function pointer access
- Flag availability of multitexturing support

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GL_ACTIVETEXTUREARB_FUNC | typedef function pointer | Signature for glActiveTextureARB extension |
| GL_CLIENTACTIVETEXTUREARB_FUNC | typedef function pointer | Signature for glClientActiveTextureARB extension |
| PFNGLCOMPRESSEDTEXIMAGE2DARBPROC | typedef function pointer | Signature for glCompressedTexImage2DARB extension |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| glActiveTextureARB_ptr | GL_ACTIVETEXTUREARB_FUNC | global | Runtime pointer to active texture function |
| glClientActiveTextureARB_ptr | GL_CLIENTACTIVETEXTUREARB_FUNC | global | Runtime pointer to client active texture function |
| glCompressedTexImage2DARB_ptr | PFNGLCOMPRESSEDTEXIMAGE2DARBPROC | global | Runtime pointer to compressed texture function |
| has_multitex | int | global | Boolean flag for multitexturing availability |

## Key Functions / Methods
None. This is a declaration/definition header only.

## Control Flow Notes
Initialization-time header. Uses conditional compilation (`#ifdef __OGL_Win32_cpp__`) to distinguish:
- **Definition context** (OGL_Win32.cpp): Initializes global pointers to NULL/0
- **Declaration context** (other files): Declares externs and provides macros for convenient access

The actual function pointers are populated at runtime by `setup_gl_extensions()` (defined elsewhere).

## External Dependencies
- `<GL/gl.h>`, `<GL/glext.h>` ΓÇö OpenGL core and extension headers
- `setup_gl_extensions()` ΓÇö external function that populates function pointers at runtime
