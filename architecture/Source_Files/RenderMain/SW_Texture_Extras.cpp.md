# Source_Files/RenderMain/SW_Texture_Extras.cpp

## File Purpose

Manages software-rendered texture opacity tables and their XML configuration. Builds per-texture opacity lookup tables based on shading data and bit depth, and provides XML parsing for texture opacity settings (type, scale, shift). Part of the software rendering pipeline for texture management.

## Core Responsibilities

- Build opacity lookup tables from shading tables for 16-bit and 32-bit color modes
- Manage texture storage across multiple collections (2D array indexed by collection and bitmap)
- Load/unload opacity tables for entire collections
- Parse XML texture definitions and apply opacity parameters
- Singleton management of global texture extras state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class (header) | Single texture with opacity type, scale, shift, and computed opacity lookup table |
| `SW_Texture_Extras` | class (singleton) | Global manager for all textures; maintains `texture_list[NUMBER_OF_COLLECTIONS]` vector-of-vectors |
| `XML_SW_Texture_Parser` | class | Parses `<texture>` XML elements; extracts collection, bitmap, opacity type/scale/shift |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SW_Texture_Extras::m_instance` | `SW_Texture_Extras*` | static | Singleton instance pointer |
| `bit_depth` | `short` (extern) | global | Current display bit depth (16 or 32); determines opacity table computation |
| `SW_Texture_Parser` | `XML_SW_Texture_Parser` | static | Reusable parser for texture elements |
| `SW_Texture_Extras_Parser` | `XML_ElementParser` | static | Root parser for software texture collection |

## Key Functions / Methods

### SW_Texture::build_opac_table

- **Signature:** `void build_opac_table()`
- **Purpose:** Compute an opacity lookup table (indexed by shading table index) and store in `m_opac_table`.
- **Inputs:** None (uses `m_shape_descriptor`, `m_opac_type`, `m_opac_scale`, `m_opac_shift`)
- **Outputs/Return:** Populates `m_opac_table` vector with 256 `uint8` opacity values
- **Side effects:** Resizes `m_opac_table` to `MAXIMUM_SHADING_TABLE_INDEXES`; reads SDL pixel format and shading tables
- **Calls:** `get_shape_bitmap_and_shading_table()`, `SDL_GetVideoSurface()`, `PIN()` (clamping macro)
- **Notes:** Branches on `bit_depth` (16 vs 32). For each shading table index, extracts RGB, applies opacity type (1=full, 2=average RGB, 3=max RGB), scales by `m_opac_scale` and `m_opac_shift`, clamps to [0, 255].

### SW_Texture_Extras::AddTexture

- **Signature:** `SW_Texture *AddTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Locate or create a texture entry for the given shape descriptor.
- **Inputs:** `ShapeDesc` (shape descriptor macro)
- **Outputs/Return:** Pointer to `SW_Texture` at `texture_list[Collection][Bitmap]`
- **Side effects:** Resizes `texture_list[Collection]` if needed
- **Calls:** `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()` (macros)
- **Notes:** Does not initialize texture properties; caller must set them.

### SW_Texture_Extras::GetTexture

- **Signature:** `SW_Texture *GetTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Retrieve texture entry, or null if out of bounds.
- **Inputs:** `ShapeDesc`
- **Outputs/Return:** Pointer to `SW_Texture` or `0` if bitmap index exceeds vector size
- **Side effects:** None
- **Calls:** `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()`
- **Notes:** Bounds-checked; safe.

### SW_Texture_Extras::Load

- **Signature:** `void Load(short collection_index)`
- **Purpose:** Build opacity tables for all textures in a collection.
- **Inputs:** `collection_index`
- **Outputs/Return:** None
- **Side effects:** Calls `build_opac_table()` on each texture in the collection
- **Calls:** `SW_Texture::build_opac_table()`
- **Notes:** Called during level load to prepare rendering data.

### SW_Texture_Extras::Unload

- **Signature:** `void Unload(short collection_index)`
- **Purpose:** Clear opacity tables for all textures in a collection.
- **Inputs:** `collection_index`
- **Outputs/Return:** None
- **Side effects:** Clears `m_opac_table` on each texture
- **Calls:** `SW_Texture::clear_opac_table()`
- **Notes:** Called during level unload or collection swap.

### XML_SW_Texture_Parser::Start / HandleAttribute / AttributesDone

- **Signature:** `bool Start()`, `bool HandleAttribute(const char *Tag, const char *Value)`, `bool AttributesDone()`
- **Purpose:** Parse a single `<texture>` XML element with attributes `coll`, `bitmap`, `opac_type`, `opac_scale`, `opac_shift`.
- **Inputs:** XML tag name, value string
- **Outputs/Return:** `true` on success, `false` on parse error
- **Side effects:** On `AttributesDone()`, creates/updates texture entry via `SW_Texture_Extras::instance()->AddTexture()` and sets properties
- **Calls:** `ReadBoundedInt16Value()`, `ReadFloatValue()` (XML helpers); `BUILD_DESCRIPTOR()`, `SW_Texture_Extras::instance()`
- **Notes:** Default values: `OpacityType=0`, `OpacityScale=1.0`, `OpacityShift=0.0`.

### SW_Texture_GetParser / SW_Texture_Extras_GetParser

- **Signature:** `XML_ElementParser *SW_Texture_GetParser()`, `XML_ElementParser *SW_Texture_Extras_GetParser()`
- **Purpose:** Return parser instances for integration into XML loading pipeline.
- **Inputs:** None
- **Outputs/Return:** Pointers to static parser instances
- **Side effects:** `SW_Texture_Extras_GetParser()` adds texture parser as a child
- **Calls:** `AddChild()`
- **Notes:** `SW_Texture_Extras_GetParser()` constructs a root `<software>` element parser.

## Control Flow Notes

This file is invoked during level initialization. The XML parsers are registered with the main game data loader. When textures are configured in a map's XML, the parsers extract opacity settings and store them in `SW_Texture_Extras` (singleton). During collection load (`Load()`), opacity tables are built from shading data. During unload or collection swap, tables are cleared (`Unload()`). The opacity table is indexed by shading table index and returned via `SW_Texture::opac_table()` for use during software rendering.

## External Dependencies

- **SDL:** `SDL_GetVideoSurface()`, `SDL_PixelFormat` (pixel format query)
- **Shapes system:** `get_shape_bitmap_and_shading_table()`, `bitmap_definition`, shape descriptor macros (`GET_COLLECTION`, `GET_DESCRIPTOR_*`, `BUILD_DESCRIPTOR`)
- **XML parsing:** `XML_ElementParser` base class, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `StringsEqual()`, `UnrecognizedTag()`
- **Utility:** `PIN()` macro (clamp to range)
- **Collection/shape definitions:** `collection_definition.h`, `shape_descriptors.h`, `MAXIMUM_SHADING_TABLE_INDEXES`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`
