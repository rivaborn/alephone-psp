# Source_Files/RenderMain/shape_descriptors.h

## File Purpose
Defines the binary layout and macros for 16-bit shape descriptors used to index 3D graphics shapes in Aleph One (Marathon engine port). Encodes collection, shape index, and color lookup table (CLUT) information into a single uint16 value.

## Core Responsibilities
- Define `shape_descriptor` type and bit-field layout (8 bits shape, 5 bits collection, 3 bits CLUT)
- Enumerate all 32 game object/scenery collections (weapons, enemies, walls, landscape, etc.)
- Provide macros to extract shape and collection indices from descriptors
- Provide macros to construct descriptors and collection values with CLUTs
- Define maximum limits for each descriptor component

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef (uint16) | Packed 16-bit index: [CLUT.3][Collection.5][Shape.8] |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a compile-time definitions header.

## Macros
| Macro | Purpose |
|-------|---------|
| `GET_DESCRIPTOR_SHAPE(d)` | Extract shape index (lower 8 bits) from descriptor |
| `GET_DESCRIPTOR_COLLECTION(d)` | Extract collection index (middle 5 bits) from descriptor |
| `BUILD_DESCRIPTOR(collection, shape)` | Construct descriptor from collection and shape indices |
| `BUILD_COLLECTION(collection, clut)` | Construct collection value with embedded CLUT |
| `GET_COLLECTION_CLUT(collection)` | Extract CLUT index from collection value |
| `GET_COLLECTION(collection)` | Extract collection index from collection value |

## Collections Enumerated
32 total: interface, weapons, enemies (juggernaut, tick, hunter, trooper, fighter, defender, yeti, enforcer, hummer, compiler, cyborg), items, walls (5 themed sets), scenery (5 themed sets), landscape (4 variations), civilian variants.

## Control Flow Notes
Purely data-definition; used by rendering and sprite management systems to decode which graphics asset to display. Not part of runtime control flow.

## External Dependencies
None. Self-contained definition header; used throughout render pipeline and sprite systems.
