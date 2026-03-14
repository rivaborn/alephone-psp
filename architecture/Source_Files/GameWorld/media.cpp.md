# Source_Files/GameWorld/media.cpp

## File Purpose

Manages in-game media (liquids/fluids) such as water, lava, goo, sewage, and Jjaro. Handles instantiation, per-frame updates, querying media properties (damage, sounds, visual effects), and serialization/XML loading of media definitions.

## Core Responsibilities

- Creating and destroying media instances in the game world
- Updating media height, texture, and flow direction each frame
- Querying media properties: damage, sounds, detonation effects, fade effects
- Checking media presence in specific environment codes
- Serializing/deserializing media data to byte streams
- Parsing and applying XML-based media definition overrides
- Counting active media slots for save-game compatibility

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | Instance of media (referenced in map.h); holds type, height, flow, origin, texture, light-binding |
| `media_definition` | struct | Type definition; referenced from `media_definitions.h`; stores collection, shape, transfer mode, damage, sounds, effects |
| `XML_LiquidParser` | class | XML parser for `<liquid>` elements; modifies media definitions at runtime |
| `XML_LqEffectParser` | class | Nested XML parser for `<effect>` children; parses detonation effect indices |
| `XML_LqSoundParser` | class | Nested XML parser for `<sound>` children; parses sound indices by event type |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | global | Dynamic array of all active media instances; indexed by `media_index` |
| `original_media_definitions` | `media_definition*` | static | Backup of original media types before XML overrides; allocated on first parse |
| `LqEffectParser` | `XML_LqEffectParser` | static | Reused parser instance for detonation effects |
| `LqSoundParser` | `XML_LqSoundParser` | static | Reused parser instance for media sounds |
| `LiquidParser` | `XML_LiquidParser` | static | Reused parser instance for individual liquid definitions |
| `LiquidsParser` | `XML_ElementParser` | static | Root parser for `<liquids>` element tree |

## Key Functions / Methods

### new_media
- **Signature:** `size_t new_media(struct media_data *initializer)`
- **Purpose:** Allocate and initialize a new media instance in the world.
- **Inputs:** Pointer to initialized `media_data` structure with type, light_index, etc.
- **Outputs/Return:** Index of the new media, or `UNONE` if no free slots.
- **Side effects:** Modifies `MediaList`; calls `update_one_media()` to compute initial height and texture.
- **Calls:** `SLOT_IS_FREE()`, `MARK_SLOT_AS_USED()`, `update_one_media()`
- **Notes:** Sets origin to (0,0); caller must set light_index before calling. Uses slot-based allocation with 0x8000 flag.

### update_medias
- **Signature:** `void update_medias(void)`
- **Purpose:** Update all active media each frame (height, texture, flow).
- **Inputs:** None (reads global `dynamic_world->tick_count`, light intensities, trig tables).
- **Outputs/Return:** None.
- **Side effects:** Modifies height, texture, origin (flow) of all active media.
- **Calls:** `update_one_media()`, sine/cosine lookup tables.
- **Notes:** Called once per tick; advances media origin by current_magnitude in current_direction.

### update_one_media
- **Signature:** `static void update_one_media(size_t media_index, bool force_update)`
- **Purpose:** Recompute height and texture for a single media instance.
- **Inputs:** Media index, force_update flag (unused in current code).
- **Outputs/Return:** None.
- **Side effects:** Updates `height` (computed from light intensity) and `texture` fields.
- **Calls:** `get_media_data()`, `get_media_definition()`, `get_light_intensity()`.
- **Notes:** Height is computed as `low + (high - low) * light_intensity`. Animated texture selection is #if 0'd out.

### get_media_data
- **Signature:** `media_data *get_media_data(const size_t media_index)`
- **Purpose:** Safely retrieve a media instance by index.
- **Inputs:** Media index.
- **Outputs/Return:** Pointer to media_data, or NULL if out of bounds or slot unused.
- **Side effects:** None.
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()`.
- **Notes:** Bounds checking; idiot-proofing for safety.

### get_media_definition
- **Signature:** `media_definition *get_media_definition(const short type)`
- **Purpose:** Retrieve type definition for a media by enum.
- **Inputs:** Media type (e.g., `_media_water`, `_media_lava`).
- **Outputs/Return:** Pointer to definition, or NULL if invalid type.
- **Side effects:** None.
- **Calls:** `GetMemberWithBounds()`.

### get_media_detonation_effect, get_media_sound, get_media_damage
- **Purpose:** Query media properties for sound/effect/damage lookups.
- **Inputs:** Media index (or media_index + type); scale for damage.
- **Outputs/Return:** Effect/sound ID or damage definition pointer.
- **Side effects:** `get_media_damage()` modifies the damage's scale field.
- **Calls:** `get_media_data()`, `get_media_definition()`.
- **Notes:** All perform NULL checks for missing media/definitions; return NONE/NULL on error.

### get_media_submerged_fade_effect, get_media_collection
- **Purpose:** Query media properties (fade effect when submerged, texture collection).
- **Inputs:** Media index.
- **Outputs/Return:** Fade effect ID or bool/collection ID.
- **Calls:** `get_media_data()`, `get_media_definition()`.

### IsMediaDangerous
- **Signature:** `bool IsMediaDangerous(short media_index)`
- **Purpose:** Check if media deals damage (e.g., lava).
- **Inputs:** Media type index.
- **Outputs/Return:** True if damage type != NONE and base damage > 0.
- **Calls:** `get_media_definition()`.
- **Notes:** Used by monster AI to avoid stepping in dangerous media.

### count_number_of_medias_used
- **Signature:** `size_t count_number_of_medias_used()`
- **Purpose:** Count the highest used media slot (for save compatibility).
- **Inputs:** None.
- **Outputs/Return:** Number of used slots (index of last + 1), or 0 if none.
- **Side effects:** None.
- **Calls:** `SLOT_IS_USED()`.
- **Notes:** Fixed countdown bug parallel to similar issue in map.cpp.

### pack_media_data, unpack_media_data
- **Signature:** `uint8 *pack_media_data(uint8 *Stream, media_data* Objects, size_t Count)` (and unpack)
- **Purpose:** Serialize/deserialize media instances to/from byte stream.
- **Inputs:** Stream pointer, object array, count.
- **Outputs/Return:** Updated stream pointer (advanced by Count * SIZEOF_media_data).
- **Side effects:** Writes to or reads from stream.
- **Calls:** `ValueToStream()`, `StreamToValue()`.
- **Notes:** Preserves binary format compatibility; handles unused padding fields.

### XML_LiquidParser methods
- **Purpose:** Parse XML `<liquid>` elements to override media definitions at runtime.
- **Start():** Initialize presence flags and backup original definitions on first call.
- **HandleAttribute():** Parse attributes (index, coll, frame, transfer, damage_freq, submerged).
- **AttributesDone():** Apply parsed values to the media definition; set up child parsers (effects, sounds, damage).
- **ResetValues():** Restore original media definitions.
- **Calls:** `Damage_SetPointer()` to attach nested damage parser.

### Liquids_GetParser
- **Signature:** `XML_ElementParser *Liquids_GetParser()`
- **Purpose:** Return the root XML parser for loading media overrides from game scripts.
- **Inputs:** None.
- **Outputs/Return:** Pointer to `LiquidsParser`.
- **Side effects:** Wires up child parsers (LiquidParser, effect parser, sound parser, damage parser).
- **Calls:** `AddChild()`.

## Control Flow Notes

**Initialization:** `Liquids_GetParser()` is called during game/level load to set up XML parsing of media definitions.

**Per-frame update:** `update_medias()` is called once per tick to advance media flow and recompute heights based on light intensity.

**Querying:** Game code and effects systems call `get_media_*` functions to look up sounds, effects, and damage for media interactions.

**Serialization:** `pack_media_data()` / `unpack_media_data()` are called during save/load to preserve media state across sessions.

## External Dependencies

- **map.h** ΓÇö `media_data` struct, `SLOT_IS_USED/MARK_SLOT_AS_USED` macros, slot allocation conventions
- **effects.h** ΓÇö `NUMBER_OF_EFFECT_TYPES` enum
- **fades.h** ΓÇö Fade effect types (e.g., `_fade_tint_blue`)
- **lightsource.h** ΓÇö `get_light_intensity()` function
- **SoundManager.h** ΓÇö Sound IDs and definitions
- **DamageParser.h** ΓÇö Damage parsing utilities
- **Packing.h** ΓÇö Serialization macros (`StreamToValue`, `ValueToStream`)
- **media_definitions.h** ΓÇö `media_definitions[]` array and `NUMBER_OF_MEDIA_TYPES` (included from header, defined elsewhere)

**Defined elsewhere:**
- `media_definitions[]` ΓÇö Master array of media type definitions
- `dynamic_world` ΓÇö Game state including tick count
- `cosine_table[]`, `sine_table[]` ΓÇö Trig lookups for flow direction
- `GetMemberWithBounds()` ΓÇö Bounds-checked array accessor
