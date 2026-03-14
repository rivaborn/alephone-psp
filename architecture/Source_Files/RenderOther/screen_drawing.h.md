# Source_Files/RenderOther/screen_drawing.h

## File Purpose
Interface header for screen drawing and rendering operations in the Aleph One game engine. Defines UI layout (rectangles, colors, fonts) and provides functions for drawing shapes, text, and manipulating the screen framebuffer. Supports both classic and SDL-based rendering backends with XML configuration for interface elements.

## Core Responsibilities
- Define rectangle IDs for game HUD and interface buttons (player name, weapon display, menu buttons, etc.)
- Define color palette indices for UI rendering (weapons, inventory, computer interface colors)
- Define font identifiers for different UI text types (interface, weapon names, computer terminal, net stats)
- Provide shape/sprite rendering primitives with source/destination clipping
- Provide text rendering with measurement and justification support
- Manage rendering "ports" (current render target context)
- Support HUD buffering and screen manipulation (erase, fill, scroll, frame)
- Enable XML-driven UI layout configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_rectangle` | struct | Portable rectangle (top, left, bottom, right) for screen coordinates |
| Rectangle IDs enum | enum | Named constants for fixed UI regions (0ΓÇô13, e.g. `_player_name_rect`, `_weapon_display_rect`) |
| Color indices enum | enum | Named constants for palette colors (energy bar colors, inventory colors, computer interface colors) |
| Text flags enum | enum | Bitflags for text rendering (center, justify, wrap, etc.) |
| Font indices enum | enum | Named constants for font types (interface, weapon name, computer terminal, etc.) |

## Global / File-Static State
None directly declared. Functions manage implicit global state (current rendering port, HUD buffer, font/color configuration) that is opaque to this header.

## Key Functions / Methods

### initialize_screen_drawing
- Signature: `void initialize_screen_drawing(void)`
- Purpose: Initialize screen drawing subsystem and default rendering state
- Inputs: None
- Outputs: None
- Side effects: Sets up rendering contexts, loads default fonts/colors
- Calls: Not visible in header

### _draw_screen_shape
- Signature: `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source)`
- Purpose: Draw a sprite/shape to screen with source clipping
- Inputs: shape descriptor (collection + shape ID), destination rectangle, source clipping rectangle (NULL = full shape bounds)
- Outputs: None (renders to current port)
- Side effects: Modifies framebuffer at current port
- Calls: Not visible in header
- Notes: Source clipping is optional; NULL uses shape's natural bounding rectangle

### _draw_screen_text
- Signature: `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color)`
- Purpose: Draw styled text to screen with justification and wrapping
- Inputs: text string, destination rectangle, justification flags (center, wrap, etc.), font index, color index
- Outputs: None (renders to current port)
- Side effects: Modifies framebuffer; may use multiple lines if wrap flag set
- Calls: Not visible in header

### _text_width (overloaded)
- Signature: `short _text_width(const char *buffer, short font_id)` and `short _text_width(const char *buffer, int start, int length, short font_id)`
- Purpose: Measure text width in pixels for layout
- Inputs: text buffer, font index, optional substring range (start, length)
- Outputs: Width in pixels
- Side effects: None
- Calls: Not visible in header
- Notes: Two overloads support full string or substring measurement

### Port management functions
- **Signature**: `void _set_port_to_screen_window(void)`, `_set_port_to_gworld(void)`, `_restore_port(void)`, `_set_port_to_term(void)` (SDL only), `_set_port_to_HUD(void)`
- **Purpose**: Switch the rendering context (target framebuffer)
- **Side effects**: Subsequent draw calls target the selected port
- **Notes**: `_restore_port()` reverts to previously active port; HUD is explicitly buffered (separate from main screen)

### Screen manipulation
- `_erase_screen(short color_index)`: Clear screen to solid color
- `_fill_screen_rectangle(...)`: Fill rectangle with color (respects current port)
- `_fill_rect(...)`, `_frame_rect(...)`: Primitive rectangle operations
- `_scroll_window(short dy, ...)`: Scroll region vertically with background fill

### Configuration queries
- `get_interface_rectangle(short index)`: Fetch screen_rectangle for named UI region
- `get_interface_color(short index)`: Fetch rgb_color for named color index
- `get_interface_font(short index)`: Fetch FontSpecifier for named font index
- `_get_font_line_height(short font_index)`: Query font metrics

### XML configuration
- `InterfaceRectangles_GetParser()`: Obtain XML_ElementParser for loading UI layout from XML
- `SetColorFontParserToScreenDrawing()`: Wire color and font parsers to use screen-drawing constants

### SDL inline utilities (ifdef SDL)
- `draw_text(...)`: Inline wrapper for font_info::draw_text with NULL check
- `text_width(...)`, `char_width(...)`, `trunc_text(...)`: Measurement helpers
- `draw_polygon(...)`, `draw_line(...)`, `draw_rectangle(...)`: Low-level geometry drawing to SDL surface

## Control Flow Notes
This header provides the rendering API layer for the game engine's UI system. Typical flow:
1. **Initialization**: `initialize_screen_drawing()` sets up default ports and fonts
2. **Frame rendering**: Switch to appropriate port (`_set_port_to_screen_window()`, `_set_port_to_HUD()`), clear with `_erase_screen()`, draw UI elements via shape and text functions
3. **Configuration**: XML parsing of interface rectangles allows runtime layout customization
4. **Shutdown**: Implicit (no shutdown function in header; managed elsewhere)

HUD is explicitly buffered (separate from main screen), suggesting double-buffering for flicker-free overlay rendering.

## External Dependencies
- **XML_ElementParser.h**: XML parsing for UI layout configuration
- **shape_descriptors.h**: Shape/sprite ID encoding (collection + shape bits)
- **sdl_fonts.h**: Font abstraction (font_info, sdl_font_info, ttf_font_info classes)
- **SDL** (conditional): Graphics library for surface operations, geometry drawing
- Standard C: stdio, strlen, sprintf (via indirect includes)
- Defined elsewhere: `FontSpecifier` class, `rgb_color` type, underlying rendering backend
