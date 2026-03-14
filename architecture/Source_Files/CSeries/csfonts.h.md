# Source_Files/CSeries/csfonts.h

## File Purpose
Header file defining font styling constants and text rendering configuration structures for the Aleph One game engine. Provides a portable abstraction for font specifications across platforms, including support for multiple font variants (normal, bold, oblique, bold-oblique) and text styling effects.

## Core Responsibilities
- Define style bit-flags for text rendering (bold, italic, underline, shadow, outline)
- Provide `TextSpec` struct for bundling font configuration (ID, size, style, path data)
- Support per-style font paths for TrueType font rendering
- Enable height adjustment for fine-tuning text layout

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TextSpec` | struct | Encapsulates complete font configuration: font ID, style flags, size, vertical adjustment, and paths to different font variants (normal, oblique, bold, bold-oblique) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `styleNormal` | const int | global | Bit-flag value (0) for normal text styling |
| `styleBold` | const int | global | Bit-flag value (1) for bold text |
| `styleItalic` | const int | global | Bit-flag value (2) for italic text |
| `styleUnderline` | const int | global | Bit-flag value (4) for underlined text |
| `styleShadow` | const int | global | Bit-flag value (16) for drop-shadow effect |
| `styleOutline` | const int | global (disabled) | Bit-flag (8) for outline/halo effect; commented outΓÇöincompatible with TTF rendering |

## Key Functions / Methods
None. This is a pure header file with type and constant definitions.

## Control Flow Notes
Not inferable from this file. This is a passive data structure definition with no procedural logic; used by font rendering and text layout subsystems throughout the engine.

## External Dependencies
- **cstypes.h**: Provides platform-specific integer types (`int16`, `uint16`)
- **\<string\>**: Standard library for `std::string` path storage in `TextSpec`

## Notes
- The commented-out `styleOutline` constant (line 33) indicates prior support for outline rendering that was removed due to TTF incompatibilityΓÇölikely when the engine migrated to TrueType font support.
- Style flags are bitwise composable (e.g., `styleBold | styleUnderline`), enabling multi-style text rendering.
- `TextSpec` holds paths to four font variants; rendering code likely selects the appropriate path based on the active `style` bits.
- `adjust_height` allows per-font baseline/vertical offset tuningΓÇöuseful for inter-font consistency.
