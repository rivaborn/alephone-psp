# Source_Files/GameWorld/media.h

## File Purpose
Defines the media (hazard/liquid) system for the game world. Media represents volumes of liquid or hazardous substances (water, lava, goo, etc.) that affect gameplay through damage, currents, sounds, and visual effects. Provides data structures, enums, and function prototypes for media creation, updating, and querying.

## Core Responsibilities
- Define media types and enumerated constants (water, lava, goo, sewage, Jjaro goo)
- Store media properties: type, height, current direction/magnitude, texture, light binding
- Manage global media list (vector-based dynamic allocation)
- Provide accessors and utilities for media queries (danger, environment membership, sound/effect lookup)
- Support serialization/deserialization of media data (pack/unpack)
- Integrate with XML-based level parsing for liquids configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | 32-byte media volume record: type, height, current, light link, texture, transfer mode |
| media types enum | enum | 5 types: `_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro` |
| media flags enum | enum | `_media_sound_obstructed_by_floor` ΓÇô sound suppression when media is under floor |
| media detonation types enum | enum | Visual effects: small/medium/large detonation, emergence |
| media sounds enum | enum | 9 event types: feet/head entering/leaving, splashing, ambient (above/below), platform events |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | extern global | Dynamic array of all media volumes in current map |
| `MAXIMUM_MEDIAS_PER_MAP` | macro | global | Expands to `MediaList.size()` (replaces old fixed limit) |

## Key Functions / Methods

### new_media
- **Signature:** `size_t new_media(struct media_data *data)`
- **Purpose:** Create and register a new media volume in the world
- **Inputs:** Pointer to initialized `media_data` structure
- **Outputs/Return:** Index/size_t of newly created media (for future lookups)
- **Side effects:** Modifies global `MediaList`; allocates memory
- **Calls:** Implicitly vector operations (push_back equivalent)
- **Notes:** Media height derived from linked light intensity; placement anchored at origin with current direction/magnitude

### update_medias
- **Signature:** `void update_medias(void)`
- **Purpose:** Per-frame or per-tick update of all media (e.g., animations, sound playback, effects)
- **Inputs:** None (uses global `MediaList`)
- **Outputs/Return:** None
- **Side effects:** Updates media state; may trigger sounds/effects; references global state
- **Calls:** (implementation not visible; likely internal)
- **Notes:** Called during world update phase

### get_media_data
- **Signature:** `media_data *get_media_data(const size_t media_index)`
- **Purpose:** Accessor for safe media record lookup
- **Inputs:** Index into `MediaList`
- **Outputs/Return:** Pointer to `media_data`, or null if out of bounds
- **Side effects:** None
- **Calls:** (bounds checking implied)
- **Notes:** Preferred over direct array access; guards invalid indices

### get_media_sound / get_media_detonation_effect / get_media_damage
- **Signature:** `short get_media_sound(short media_index, short type)`; similar for others
- **Purpose:** Query audio/visual/damage properties of a media type for gameplay events
- **Inputs:** Media index; event/effect type enum
- **Outputs/Return:** Sound/effect/damage definition index (or NONE)
- **Side effects:** None (read-only queries)
- **Notes:** Used by physics/audio systems to play appropriate effects when player interacts with media

### IsMediaDangerous
- **Signature:** `bool IsMediaDangerous(short media_type)`
- **Purpose:** Determine if a media type inflicts damage or blocks movement (used by monsters/AI pathfinding)
- **Inputs:** Media type enum value
- **Outputs/Return:** Boolean
- **Side effects:** None
- **Notes:** Guides monster behavior (e.g., avoid lava, accept water)

### pack_media_data / unpack_media_data
- **Signature:** `uint8 *pack_media_data(uint8 *Stream, media_data* Objects, size_t Count)`; reverse for unpack
- **Purpose:** Serialize/deserialize media records to/from byte streams (for save files, network transmission)
- **Inputs:** Byte stream, array of media objects, count
- **Outputs/Return:** Updated stream pointer (points after written/read data)
- **Side effects:** Modifies stream buffer; may update media array
- **Notes:** Records are 32 bytes fixed size; supports multiple objects in one call

### Liquids_GetParser
- **Signature:** `XML_ElementParser *Liquids_GetParser()`
- **Purpose:** Return XML parser for liquid/media configuration from external files (map definitions)
- **Inputs:** None
- **Outputs/Return:** Pointer to configured XML_ElementParser for "liquids" element
- **Side effects:** None (returns reference to parser; parsing done elsewhere)
- **Notes:** Enables data-driven media customization in level/scenario files

## Control Flow Notes
**Initialization:** Media is typically loaded from map data during `initialize_map_for_new_level()` (called from map.h context).

**Per-Frame:** `update_medias()` is invoked during world updates to animate liquid heights, trigger sounds, and apply effects.

**Runtime Queries:** Physics and audio systems call `get_media_*` functions on collision/entry events; monster AI calls `IsMediaDangerous()` during pathfinding.

**Persistence:** Media state is saved/restored via pack/unpack routines when saving/loading games or handling network replication.

## External Dependencies
- **map.h** ΓÇô `SLOT_IS_USED` macro for media slot validity testing; constants like `WORLD_ONE`, `MAXIMUM_VERTICES_PER_POLYGON`
- **XML_ElementParser.h** ΓÇô `XML_ElementParser` base class for data-driven parsing
- **\<vector\>** ΓÇô STL dynamic array for `MediaList`
- **csmacros.h** (implied via includes) ΓÇô `TEST_FLAG16`, `SET_FLAG16` macros for bit manipulation
