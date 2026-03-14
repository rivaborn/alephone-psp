# Source_Files/RenderMain/AnimatedTextures.cpp

## File Purpose
Implements animated wall textures for the Aleph One game engine by managing texture frame sequences and their timing. Provides XML-configurable animation sequences that translate texture descriptors at runtime, allowing seamless texture animation in level rendering.

## Core Responsibilities
- Manage animation sequences as frame lists with independent timing (advance/reverse)
- Track animation state per sequence (current frame and tick phase)
- Translate texture descriptors to animated frames during rendering
- Parse XML configuration to define animation sequences per collection
- Maintain per-collection animation registry for efficient lookup during frame rendering

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `AnimTxtr` | class | Single animation sequence: frame list, timing (ticks-per-frame), phase state, and optional texture selector |
| `XML_AT_ClearParser` | class | XML parser for `<clear>` elements to reset animation sequences |
| `XML_AT_FrameParser` | class | XML parser for `<frame>` elements defining a single frame index |
| `XML_AT_SequenceParser` | class | XML parser for `<sequence>` elements defining a complete animation with frames, timing, and optional phase offsets |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AnimTxtrList` | `static vector<AnimTxtr>[NUMBER_OF_COLLECTIONS]` | static | Stores all animation sequences indexed by collection ID for fast lookup |
| `TempFrameList` | `static vector<short>` | static | Temporary storage for frame indices during XML parsing |
| `AT_ClearParser` | `static XML_AT_ClearParser` | static | Singleton XML parser instance for `<clear>` elements |
| `AT_FrameParser` | `static XML_AT_FrameParser` | static | Singleton XML parser instance for `<frame>` elements |
| `AT_SequenceParser` | `static XML_AT_SequenceParser` | static | Singleton XML parser instance for `<sequence>` elements |
| `AnimatedTexturesParser` | `static XML_ElementParser` | static | Root XML parser for `<animated_textures>` element |

## Key Functions / Methods

### AnimTxtr::Load
- **Signature:** `void Load(vector<short>& _FrameList)`
- **Purpose:** Populate this animation with a new frame sequence.
- **Inputs:** Reference to a frame-index vector.
- **Outputs/Return:** None.
- **Side effects:** Swaps internal `FrameList`; wraps `FramePhase` to valid range.
- **Calls:** `FrameList.swap()`, modulo operator.
- **Notes:** Uses `swap()` for efficient transfer; normalizes `FramePhase` to prevent out-of-bounds access.

### AnimTxtr::Translate
- **Signature:** `bool Translate(short& Frame)`
- **Purpose:** Map a texture frame ID through the animation, returning the current animated frame.
- **Inputs:** Frame ID (modified in place if translation succeeds).
- **Outputs/Return:** `true` if frame was in the animation sequence, `false` otherwise.
- **Side effects:** Modifies input `Frame` to the current animated frame.
- **Calls:** Linear search through `FrameList` if no selector is set.
- **Notes:** If `Select >= 0`, only translates that specific texture; otherwise translates any frame found in the sequence. Applies `FramePhase` offset and wraps.

### AnimTxtr::SetTiming
- **Signature:** `void SetTiming(short _NumTicks, size_t _FramePhase, size_t _TickPhase)`
- **Purpose:** Configure animation speed and initial phase state.
- **Inputs:** Ticks per frame (negative = reverse), frame phase, tick phase.
- **Outputs/Return:** None.
- **Side effects:** Updates `NumTicks`, `FramePhase`, `TickPhase`; normalizes phases to valid ranges.
- **Calls:** Integer division and modulo for phase bounds correction.
- **Notes:** Handles tick-phase overflow by rolling into frame-phase; wraps frame-phase to frame count.

### AnimTxtr::Update
- **Signature:** `void Update()`
- **Purpose:** Advance animation state by one tick.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Increments or decrements `TickPhase` and `FramePhase` based on `NumTicks` sign; wraps phases.
- **Calls:** None (inline state manipulation).
- **Notes:** Handles forward (positive `NumTicks`) and reverse (negative `NumTicks`) animation directions; wraps frame phase circularly.

### AnimTxtr_Update
- **Signature:** `void AnimTxtr_Update()`
- **Purpose:** Update all animated textures across all collections.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `Update()` on every `AnimTxtr` in `AnimTxtrList`.
- **Calls:** `AnimTxtr::Update()` for each sequence.
- **Notes:** Called once per frame tick; iterates all collections and all sequences within each.

### AnimTxtr_Translate
- **Signature:** `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- **Purpose:** Render-path texture translation: convert a shape descriptor to its current animated frame.
- **Inputs:** Shape descriptor (collection + frame + color table encoded).
- **Outputs/Return:** Updated shape descriptor with animated frame, or `UNONE` if invalid.
- **Side effects:** None (purely functional lookup).
- **Calls:** `GET_DESCRIPTOR_SHAPE`, `GET_DESCRIPTOR_COLLECTION`, macro decoders; `AnimTxtr::Translate()`; `get_number_of_collection_frames()`.
- **Notes:** Returns `UNONE` if texture is already `UNONE`, or if frame is out of bounds after translation. Extracts collection, frame, and color table; reconstructs descriptor with translated frame.

### AnimatedTextures_GetParser
- **Signature:** `XML_ElementParser *AnimatedTextures_GetParser()`
- **Purpose:** Provide the XML parser hierarchy for loading animated-texture configuration.
- **Inputs:** None.
- **Outputs/Return:** Pointer to root `AnimatedTexturesParser`.
- **Side effects:** Wires up child parsers (`AT_ClearParser`, `AT_SequenceParser`); `AT_SequenceParser` gets `AT_FrameParser` as child.
- **Calls:** `AddChild()` on parser objects.
- **Notes:** Called during engine initialization to register the XML parser for `<animated_textures>` blocks.

**Helper/Utility methods** (not detailed):
- `AnimTxtr::GetNumFrames()`, `GetFrame()`, `Clear()` ΓÇô basic accessors and list clearance.
- `ATDelete(int c)`, `ATDeleteAll()` ΓÇô static helpers to clear sequences for a collection or all collections.
- XML parser methods (`Start`, `HandleAttribute`, `AttributesDone`, `End`, `ResetValues`) ΓÇô standard XML parsing interface.

## Control Flow Notes
- **Initialization:** `AnimatedTextures_GetParser()` is called to register XML parsers; XML configuration populates `AnimTxtrList`.
- **Per-frame update:** `AnimTxtr_Update()` is called to tick all animations forward.
- **Rendering:** For each wall/floor texture, `AnimTxtr_Translate()` maps it to the current frame before lookup.
- **Shutdown:** Implicit via collection cleanup; `ATDeleteAll()` can be called to reset.

## External Dependencies
- **Headers:** `<vector>`, `cseries.h`, `AnimatedTextures.h`, `interface.h`
- **External symbols (defined elsewhere):**
  - `shape_descriptor`, `GET_DESCRIPTOR_SHAPE`, `GET_DESCRIPTOR_COLLECTION`, `GET_COLLECTION`, `GET_COLLECTION_CLUT`, `BUILD_DESCRIPTOR`, `BUILD_COLLECTION`, `UNONE` ΓÇô from shape_descriptors.h
  - `NUMBER_OF_COLLECTIONS` ΓÇô constant
  - `get_number_of_collection_frames()` ΓÇô from interface.h
  - `XML_ElementParser` ΓÇô base class for XML parsing framework
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadUInt16Value()` ΓÇô XML utility functions (likely in cseries or XML module)
