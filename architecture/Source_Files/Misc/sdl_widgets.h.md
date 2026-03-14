# Source_Files/Misc/sdl_widgets.h

## File Purpose
Defines a comprehensive widget library for SDL-based dialog UI construction. Provides base widget class and ~25 derived widget classes (buttons, text entry, sliders, lists, toggles, file choosers) used throughout the Aleph One engine for user interface dialogs.

## Core Responsibilities
- Define abstract `widget` base class and core widget lifecycle (draw, event handling, enable/disable)
- Provide text display widgets (labels, static text, pictures)
- Provide input widgets (text entry, number entry, password entry, key binding)
- Provide selection widgets (toggles, dropdowns, sliders, color pickers)
- Provide list widgets (generic template-based lists, level lists, string lists, metaserver lists)
- Implement callback mechanisms via `boost::function` for button clicks and selection changes
- Provide wrapper/adapter classes (`SDLWidgetWidget` and derived) for uniform widget interface
- Support widget identification and lookup by ID within dialogs

## Key Types / Data Structures
| Name | Kind | Purpose |
| --- | --- | --- |
| widget | class | Abstract base for all UI widgets; manages rect, font, state (active/dirty/enabled) |
| w_static_text | class | Non-interactive text display; owns string copy |
| w_label | class | Text label that can activate associated widget on click |
| w_button_base | class | Button with text, callback, visual states (default/active/pressed/disabled) |
| w_toggle | class | Two-state toggle (on/off) based on w_select |
| w_enabling_toggle | class | Toggle that enables/disables dependent widgets based on state |
| w_text_entry | class | Single-line editable text field with enter/value-changed callbacks |
| w_select | class | Multi-option selection widget; supports stringset labels and callbacks |
| w_slider | class | Horizontal slider with draggable thumb; numeric item selection |
| w_list_base | class | Scrollable list base with selection and item height abstraction |
| w_list<T> | template class | Generic list for arbitrary item type T with custom draw_item |
| w_items_in_room<T> | template class | Metaserver-style list with item-clicked callback |
| w_games_in_room | class | Specialized list displaying game entries with multi-line rendering |
| w_players_in_room | class | Specialized list displaying player info with color swatches |
| w_colorful_chat | class | Chat message list with colored sender names |
| ControlHitCallback | typedef | `boost::function<void(void)>` for control activation |
| selection_changed_callback_t | typedef | `boost::function<void(w_select*)>` for selection change notification |

## Global / File-Static State
None.

## Key Functions / Methods

### widget::set_enabled
- **Signature:** `void set_enabled(bool inEnabled)`
- **Purpose:** Enable/disable user interaction with widget; widget alters visual appearance when disabled
- **Inputs:** `inEnabled` ΓÇô true to allow interaction, false to disable
- **Outputs/Return:** None
- **Side effects:** Sets `enabled` member; widget::draw() responsibility to render disabled state
- **Calls:** None visible
- **Notes:** Dialog class assumes widget alters drawing behavior for disabled state

### widget::draw
- **Signature:** `virtual void draw(SDL_Surface *s) const = 0`
- **Purpose:** Render widget to SDL surface; pure virtual, must be implemented by all subclasses
- **Inputs:** `s` ΓÇô SDL surface to draw into
- **Outputs/Return:** None
- **Side effects:** Direct pixel writes to surface
- **Calls:** (implementation-dependent; typically font->draw_text, get_theme_image, etc.)
- **Notes:** Called by dialog during render pass; responsible for visual state (disabled, active, etc.)

### w_text_entry::set_text / get_text
- **Signature:** `void set_text(const char *text)` / `const char *get_text(void)`
- **Purpose:** Update or retrieve current text buffer contents
- **Inputs/Outputs:** C-string pointer (max_chars limit enforced)
- **Side effects:** Modifies internal buffer; set_text triggers value_changed_callback if registered
- **Calls:** `modified_text()`
- **Notes:** Text ownership is widget's responsibility; destructor frees buffer

### w_select::set_selection
- **Signature:** `void set_selection(size_t selection, bool simulate_user_input = false)`
- **Purpose:** Change currently selected option; optionally trigger selection_changed callback
- **Inputs:** `selection` ΓÇô index into labels array; `simulate_user_input` ΓÇô fire callback if true
- **Outputs/Return:** None
- **Side effects:** Updates `selection` member; may invoke `selection_changed_callback`
- **Calls:** `selection_changed()` if conditions met
- **Notes:** `UNONE` means unknown selection; callback behavior depends on simulate_user_input flag

### w_text_entry::set_enter_pressed_callback
- **Signature:** `void set_enter_pressed_callback(Callback func)`
- **Purpose:** Register callback to invoke when user presses Enter key
- **Inputs:** `func` ΓÇô `boost::function<void(w_text_entry*)>` callback
- **Outputs/Return:** None
- **Side effects:** Stores callback for later invocation
- **Calls:** None
- **Notes:** Callback fired by event() handler on SDL_KEYDOWN Enter event

### w_list_base::set_selection
- **Signature:** `void set_selection(size_t s)` (protected)
- **Purpose:** Change selected item in list; updates scroll position
- **Inputs:** `s` ΓÇô new selection index
- **Outputs/Return:** None
- **Side effects:** Updates `selection`, `top_item`, `thumb_y`; marks widget dirty
- **Calls:** `center_item()`
- **Notes:** Manages scrollbar thumb and visible range

### dialog::draw_dirty_widgets
- **Signature:** `void draw_dirty_widgets() const` (from sdl_dialogs.h)
- **Purpose:** Optimization: redraw only widgets marked dirty, not entire dialog
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Redraws widgets where `is_dirty()` returns true; clears dirty flag
- **Calls:** (dialog internals)
- **Notes:** Called automatically by event loop; user code can call for timer-driven updates

### w_enabling_toggle::selection_changed
- **Signature:** `void selection_changed()` (protected, override)
- **Purpose:** Update enabled state of dependent widgets when toggle state changes
- **Inputs:** None (uses `selection` member)
- **Outputs/Return:** None
- **Side effects:** Calls `set_enabled()` on each widget in `dependents` set
- **Calls:** `update_widget_enabled()`
- **Notes:** Fires when toggle selection changes; updates each dependent synchronously

## Control Flow Notes
Widget system is **event-driven**. Lifecycle:
- **Creation:** Widgets constructed and added to dialog via `dialog::add(widget*)`
- **Layout:** `dialog::layout()` called; calls `widget::place()` with SDL_Rect and placement flags
- **Event loop:** `dialog::process_events()` dispatches SDL events to active/mouse widgets
- **Rendering:** `dialog::draw()` or `draw_dirty_widgets()` calls `widget::draw()` for each widget
- **Callbacks:** User interactions trigger callbacks (`w_button` click ΓåÆ action_proc; `w_select` change ΓåÆ selection_changed_callback)
- **Destruction:** Dialog destructor cleans up widgets

No explicit update/tick cycle; widgets are stateless except for UI state (selection, text, position).

## External Dependencies
- **SDL:** `SDL.h`, `SDL_Surface`, `SDL_Rect`, `SDL_Event`, `SDLKey`
- **Boost:** `boost::function`, `boost::bind` (callback and function binding)
- **Local headers:**
  - `sdl_dialogs.h` ΓÇô `dialog` class, `placeable` interface, theme functions (`get_theme_font`, `get_theme_color`, `get_theme_space`)
  - `sdl_fonts.h` ΓÇô `font_info` class for text rendering
  - `screen_drawing.h` ΓÇô `draw_text()` inline helpers, `rgb_color`, `RGBColor`
  - `map.h` ΓÇô `entry_point` struct (for `w_levels` template specialization)
  - `tags.h` ΓÇô `Typecode` enum (for `w_file_chooser`)
  - `FileHandler.h` ΓÇô `FileSpecifier` class (for `w_file_chooser`)
  - `metaserver_messages.h` ΓÇô `GameListMessage::GameListEntry`, `MetaserverPlayerInfo` (for metaserver list widgets)
  - `network.h` ΓÇô `prospective_joiner_info` (for joining player list)
  - `binders.h` ΓÇô binding utilities (likely for callback wrapping)

**Symbols defined elsewhere:**
- `dialog` (placeable base, dialog class)
- `font_info`, `sdl_font_info` (font rendering)
- `get_theme_font()`, `get_theme_color()`, `get_theme_space()`, `get_theme_image()` (theme system)
- `draw_text()` (screen drawing)
- `entry_point`, `GameListMessage`, `MetaserverPlayerInfo`, `prospective_joiner_info` (game data)
