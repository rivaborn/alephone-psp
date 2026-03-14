# Expat/xmltok/nametab.h

## File Purpose
Provides pre-computed character classification tables for XML name validation. These static lookup tables enable efficient, spec-compliant validation of which Unicode characters are valid in XML names and which can start names, used by the Expat XML tokenizer.

## Core Responsibilities
- Encode XML specification rules for valid name characters as compact bitmap arrays
- Provide nameStart character classification (page-indexed lookup)
- Provide name continuation character classification (page-indexed lookup)
- Support sparse Unicode coverage using a paging indirection scheme to minimize memory footprint

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `namingBitmap` | static unsigned[] | 32-bit bitmap array covering ~16 Unicode "pages"; each bit indicates if a character is valid in XML names |
| `nmstrtPages` | static unsigned char[] | Page map (256 entries); maps high byte of codepoint to index for name-start character validation |
| `namePages` | static unsigned char[] | Page map (256 entries); maps high byte of codepoint to index for name-continuation character validation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `namingBitmap` | unsigned[] (96 entries) | static | Bit vector encoding valid name characters across multiple Unicode pages |
| `nmstrtPages` | unsigned char[] (256 entries) | static | Indirection table for name-start character lookup |
| `namePages` | unsigned char[] (256 entries) | static | Indirection table for name-continuation character lookup |

## Key Functions / Methods
None (data-only header).

## Control Flow Notes
Not applicable. This file is a compile-time data table included by tokenizer implementations. Lookup occurs during XML parsing when validating token names. The paging scheme allows O(1) character classification: index into `nmstrtPages` or `namePages` using the high byte, then use the returned page index to select a 32-bit word in `namingBitmap`, then test the appropriate bit within that word.

## External Dependencies
- None (no includes, no function calls)
- Included by other modules in `Expat/xmltok/` to implement character classification functions

## Notes
- Tables are pre-generated (likely from a Unicode database or specification), not computed at runtime
- The paging indirection (256-entry page tables ΓåÆ sparse bitmap regions) is a space optimization for Unicode coverage
- Valid only for XML 1.0 name rules; includes ASCII alphanumerics, common punctuation (`:`, `-`, `.`, `_`), and non-ASCII letter/digit ranges per XML spec
