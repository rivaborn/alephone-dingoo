# Source_Files/RenderMain/AnimatedTextures.cpp

## File Purpose
Implements animated wall textures for the Aleph One engine, reading sequences from XML configuration files. Maintains per-collection animation state (frame lists, timing, phases) and provides translation services to swap static texture descriptors with current animated frames during rendering.

## Core Responsibilities
- Manage animated texture sequences organized by collection ID
- Track and update animation state (current frame phase, tick phase within frame)
- Translate texture descriptors to current animated frame during rendering
- Parse XML configuration files to define animation sequences
- Support selective animation where only specified textures in a sequence are translated

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AnimTxtr` | class | Encapsulates a single animated texture sequence: frame list, timing, current phases, and selection filter |
| `XML_AT_ClearParser` | class | Parses `<clear>` XML elements to delete animations for a collection |
| `XML_AT_FrameParser` | class | Parses `<frame>` XML elements with frame index, accumulates into `TempFrameList` |
| `XML_AT_SequenceParser` | class | Parses `<sequence>` XML elements to create complete animation sequences with timing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AnimTxtrList` | `vector<AnimTxtr>[NUMBER_OF_COLLECTIONS]` | static | Per-collection storage of all animated texture sequences; indexed by collection ID |
| `TempFrameList` | `vector<short>` | static | Temporary accumulator for frame indices during XML parsing of a sequence |
| `AT_ClearParser`, `AT_FrameParser`, `AT_SequenceParser`, `AnimatedTexturesParser` | XML parser instances | static | Singleton XML element parsers registered in hierarchy |

## Key Functions / Methods

### AnimTxtr::Load
- **Signature:** `void Load(vector<short>& _FrameList)`
- **Purpose:** Transfer frame data into this animation sequence
- **Inputs:** Reference to frame list vector (ownership transferred via swap)
- **Outputs/Return:** None
- **Side effects:** Clears incoming vector; normalizes `FramePhase` to valid range
- **Calls:** `vector::swap()`, `vector::empty()`, `vector::size()`
- **Notes:** Uses move-semantics via swap; ensures phase consistency if frame list is not empty

### AnimTxtr::Translate
- **Signature:** `bool Translate(short& Frame)`
- **Purpose:** Translate a frame ID by substituting with current animated frame; returns success
- **Inputs:** Frame ID to translate (modified in place)
- **Outputs/Return:** `true` if frame was in sequence and translated; `false` otherwise
- **Side effects:** Modifies input `Frame` parameter
- **Calls:** `vector::size()`
- **Notes:** If `Select >= 0`, only translates matching frame; otherwise searches entire frame list and uses first match

### AnimTxtr::SetTiming
- **Signature:** `void SetTiming(short _NumTicks, size_t _FramePhase, size_t _TickPhase)`
- **Purpose:** Configure animation timing (ticks per frame) and current animation phase
- **Inputs:** Ticks per frame (negative for reverse), frame phase (which frame), tick phase (position within frame)
- **Outputs/Return:** None
- **Side effects:** Updates `NumTicks`, `FramePhase`, `TickPhase`; normalizes phases to valid ranges
- **Calls:** `vector::size()`
- **Notes:** Handles tick-to-frame overflow/underflow conversion; wraps phases modulo frame count

### AnimTxtr::Update
- **Signature:** `void Update()`
- **Purpose:** Advance animation state by one tick for forward or reverse playback
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Increments or decrements `TickPhase`; advances `FramePhase` when tick wraps; handles bidirectional wrapping
- **Calls:** `vector::size()`
- **Notes:** Negative `NumTicks` triggers reverse animation; wraps frame indices correctly in both directions

### AnimTxtr_Update (global)
- **Signature:** `void AnimTxtr_Update()`
- **Purpose:** Update all animation sequences across all collections; called once per frame
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Update()` on every `AnimTxtr` in every collection
- **Calls:** `AnimTxtr::Update()` for all sequences
- **Notes:** No-op if no animations loaded; engine integration point

### AnimTxtr_Translate (global)
- **Signature:** `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- **Purpose:** Main API entry point; translates a texture descriptor to its current animated frame
- **Inputs:** Shape descriptor (packed collection + frame ID)
- **Outputs/Return:** Updated descriptor with animated frame; `UNONE` if invalid/out-of-range
- **Side effects:** None (reads only)
- **Calls:** `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `AnimTxtr::Translate()`, `get_number_of_collection_frames()`, `BUILD_DESCRIPTOR()`
- **Notes:** Extracts collection and frame from descriptor; searches animation list; validates result frame before returning; used during rendering pipeline

### AnimatedTextures_GetParser
- **Signature:** `XML_ElementParser *AnimatedTextures_GetParser()`
- **Purpose:** Return root XML parser for animated texture configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to `AnimatedTexturesParser` with child parsers registered
- **Side effects:** Registers `AT_ClearParser` and `AT_SequenceParser` as children; registers `AT_FrameParser` as child of `AT_SequenceParser`
- **Calls:** `AddChild()` on parser instances
- **Notes:** Called once during engine initialization to set up XML parsing hierarchy

## Control Flow Notes
**Initialization:** `AnimatedTextures_GetParser()` registers XML hierarchy; XML configuration populates `AnimTxtrList[collection]` with `AnimTxtr` sequences.

**Per-frame update:** `AnimTxtr_Update()` is called in main loop to advance all animation phases.

**Rendering:** `AnimTxtr_Translate()` is called for each texture descriptor encountered, translating static frame IDs to current animated frames before geometry rendering.

**XML Parsing Flow:** Root element `<animated_textures>` contains `<clear>` (reset collection) and `<sequence>` (define animation) children. Each `<sequence>` contains `<frame>` children accumulating into `TempFrameList`.

## External Dependencies
- `<vector>` ΓÇö STL dynamic array (frame lists, animation collections)
- `cseries.h` ΓÇö Core utilities (types, macros, string functions)
- `AnimatedTextures.h` ΓÇö Public interface declarations
- `interface.h` ΓÇö Engine symbols: `shape_descriptor` type, macros (`GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`), `get_number_of_collection_frames()`, `UNONE`, `NUMBER_OF_COLLECTIONS`
- Base class `XML_ElementParser` ΓÇö XML parsing framework (defined elsewhere)
- Helper functions `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadUInt16Value()` ΓÇö Assumed from cseries/XML utilities
