# Expat/xmltok/nametab.h
## File Purpose
Provides pre-computed character classification tables for XML name validation. These static lookup tables enable efficient, spec-compliant validation of which Unicode characters are valid in XML names and which can start names, used by the Expat XML tokenizer.

## Core Responsibilities
- Encode XML specification rules for valid name characters as compact bitmap arrays
- Provide nameStart character classification (page-indexed lookup)
- Provide name continuation character classification (page-indexed lookup)
- Support sparse Unicode coverage using a paging indirection scheme to minimize memory footprint

## External Dependencies
- None (no includes, no function calls)
- Included by other modules in `Expat/xmltok/` to implement character classification functions



