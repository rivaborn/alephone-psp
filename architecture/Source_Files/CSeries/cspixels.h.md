# Source_Files/CSeries/cspixels.h

## File Purpose
Defines pixel type aliases and bitwise conversion macros for RGB color values in 16-bit and 32-bit pixel formats. Provides low-level support for multi-format pixel manipulation in the Aleph One game engine.

## Core Responsibilities
- Define typed aliases for pixel values at different color depths (8, 16, 32-bit)
- Establish maximum color component values for each pixel format
- Provide macros to pack RGB values (0x0000ΓÇô0xFFFF range) into pixel formats
- Provide macros to extract individual color channels (R, G, B) from packed pixels
- Document color component ranges and format specifications

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| pixel8 | typedef | 8-bit indexed color pixel |
| pixel16 | typedef | 16-bit RGB pixel (5:5:5 + 1 unused) |
| pixel32 | typedef | 32-bit ARGB pixel (8:8:8 per component) |

## Global / File-Static State
None.

## Key Functions / Methods
None. (Header contains only type definitions and bitwise conversion macros.)

### RGBCOLOR_TO_PIXEL16(r, g, b)
- **Purpose:** Pack three 16-bit RGB input values into a 16-bit pixel using 5:5:5 format
- **Inputs:** r, g, b each in range 0x0000ΓÇô0xFFFF
- **Outputs/Return:** 16-bit pixel with layout: RRRRRGGG GGBBBBBB
- **Notes:** Shifts and masks: r>>1 (11 bitsΓåÆ10-6), g>>6 (16 bitsΓåÆ5-1), b>>11 (16 bitsΓåÆ0). Input range mismatch (16-bit in, 5-bit out) requires careful caller scaling

### RED16(p), GREEN16(p), BLUE16(p)
- **Purpose:** Extract individual color components from a 16-bit pixel
- **Inputs:** p = 16-bit pixel
- **Outputs/Return:** 5-bit component (0x00ΓÇô0x1F)
- **Notes:** Reverses the packing via right-shift and mask

### RGBCOLOR_TO_PIXEL32(r, g, b)
- **Purpose:** Pack three 16-bit RGB input values into a 32-bit pixel using 8:8:8 format
- **Inputs:** r, g, b each in range 0x0000ΓÇô0xFFFF
- **Outputs/Return:** 32-bit pixel with layout: 0xRRGGBB00 or similar (RGB in upper bytes)
- **Notes:** r<<8 (bits 16ΓÇô23), g (bits 8ΓÇô15), b>>8 (bits 0ΓÇô7)

### RED32(p), GREEN32(p), BLUE32(p)
- **Purpose:** Extract individual color components from a 32-bit pixel
- **Inputs:** p = 32-bit pixel
- **Outputs/Return:** 8-bit component (0x00ΓÇô0xFF)
- **Notes:** Reverses packing via right-shift and mask

## Control Flow Notes
Not applicable. Header-only; no control flow or initialization.

## External Dependencies
- `cstypes.h` ΓÇö provides uint8, uint16, uint32 type definitions (platform-specific)

---

**Observations:**
- **16-bit format:** The 5:5:5 layout wastes 1 high bit per channel; designed for older 16-bit graphics subsystems.
- **Input/output mismatch:** Combiner macros accept 16-bit values but only use the high 5 bits; extractors return 5-bit or 8-bit values. Caller must scale appropriately if round-tripping is needed.
- **No bounds checking:** Macros assume valid input ranges; overflow silently clips/wraps.
