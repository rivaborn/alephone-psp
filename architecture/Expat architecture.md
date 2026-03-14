# Subsystem Overview

## Purpose
Provides pre-computed character classification lookup tables for the Expat XML tokenizer to efficiently validate which Unicode characters are valid in XML names and which can start names. These static bitmap arrays encode XML specification rules in a memory-efficient format suitable for the PSP's constrained 32MB RAM environment.

## Key Files
| File | Role |
|------|------|
| `Expat/xmltok/nametab.h` | Static lookup tables for XML name character classification; page-indexed bitmaps for nameStart and name continuation rules |

## Core Responsibilities
- Encode XML specification rules for valid name characters as compact bitmap arrays
- Provide nameStart character classification via page-indexed lookup
- Provide name continuation character classification via page-indexed lookup
- Support sparse Unicode coverage using paging indirection to minimize memory footprint
- Enable fast character-by-character validation during tokenization without dynamic computation

## Key Interfaces & Data Flow
**Exposes:**
- Static bitmap arrays (`nameStart`, `name` continuation tables) indexed by Unicode page and offset

**Consumed by:**
- Other modules in `Expat/xmltok/` that implement XML tokenizer character validation functions

## Runtime Role
Consulted during XML parsing at tokenization time whenever the parser needs to validate element names, attribute names, or other XML name tokens. No dynamic initialization required; tables are compile-time constants.

## Notable Implementation Details
- Paging indirection scheme trades small lookup table overhead for significant memory savings across sparse Unicode ranges
- Bitmap encoding allows multiple character classifications per byte
- Compile-time generation ensures zero runtime overhead
