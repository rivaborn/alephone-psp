# Source_Files/Misc/sdl_dialogs.h

## File Purpose
Defines SDL-based dialog and widget layout system for the Aleph One game engine. Provides classes for managing modal/modeless dialogs, widget placement/layout strategies, and theme-driven appearance (colors, fonts, images, spacing).

## Core Responsibilities
- **Dialog management**: lifecycle (start, process events, finish), widget management, event routing
- **Widget placement**: layout strategies (vertical, horizontal, table, tab-based) with flexible alignment/spacing
- **Theme system**: query and apply colors, fonts, images, and spacing metrics from loaded themes
- **Focus/input handling**: activate/deactivate widgets, process keyboard/mouse events
- **Dirty widget optimization**: redraw only widgets marked as needing updates
- **Sound effects**: play dialog-related sound events (intro, OK, cancel, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `placeable` | abstract class | Base for any UI element that can be positioned/sized within a layout |
| `widget_placer` | abstract class | Base for layout managers that own and arrange child placeables |
| `tab_placer` | class | Tabbed layout: manages multiple tabs, shows one at a time |
| `table_placer` | class | Grid layout: arranges children in columns/rows with configurable spacing and width balancing |
| `vertical_placer` | class | Vertical stack layout: arranges children top-to-bottom |
| `horizontal_placer` | class | Horizontal stack layout: arranges children left-to-right |
| `dialog` | class | Main dialog window: manages widgets, events, drawing, and modal/modeless operation |
| `placement_flags` | enum | Alignment modes (left/center/right/fill) for widgets within a layout |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `top_dialog` | `dialog*` | extern global | Points to currently active top-level dialog (NULL if none) |
| `sKeyRepeatActive` | bool | static class member (dialog) | Tracks SDL key-repeat state (SDL doesn't store this) |

## Key Functions / Methods

### dialog::run
- **Signature:** `int run(bool intro_exit_sounds = true)`
- **Purpose:** Execute dialog modally; blocks until user closes it
- **Inputs:** Flag to play intro/exit sounds
- **Outputs/Return:** Dialog result code (set via `quit()`)
- **Side effects:** Saves/restores cursor visibility, unicode input state, key-repeat state; manages global `top_dialog` pointer
- **Calls:** `start()`, `process_events()`, `finish()` (orchestrates modal loop)
- **Notes:** Modal blocking loop; restores SDL input state on exit

### dialog::start
- **Signature:** `void start(bool play_sound = true)`
- **Purpose:** Place dialog on screen; begins non-modal operation
- **Inputs:** Flag to play intro sound
- **Outputs/Return:** None
- **Side effects:** Sets `done = false`, saves cursor/input state, calls `layout()` and `draw()`
- **Calls:** `layout()`, `draw()`, sound functions
- **Notes:** Does not block; call `process_events()` and `finish()` manually for non-modal use

### dialog::process_events
- **Signature:** `bool process_events()`
- **Purpose:** Handle one batch of pending SDL events; return whether dialog is done
- **Inputs:** None (reads from SDL event queue)
- **Outputs/Return:** `true` if dialog is finished, `false` otherwise
- **Side effects:** Updates `active_widget`, `mouse_widget`, `done` flag; calls widget event handlers
- **Calls:** `event()` (private), widget event methods
- **Notes:** Should be called repeatedly in non-modal dialogs; `run()` calls this in a loop

### dialog::finish
- **Signature:** `int finish(bool play_sound = true)`
- **Purpose:** Remove dialog from screen and return result
- **Inputs:** Flag to play cancel sound
- **Outputs/Return:** Dialog result code
- **Side effects:** Removes from screen, restores cursor/input state, cleans up frame surfaces, deactivates parent dialog
- **Calls:** `draw()`, sound functions
- **Notes:** Pairs with `start()`; safe to call even if dialog never fully started

### dialog::quit
- **Signature:** `void quit(int result)`
- **Purpose:** Request dialog to close with given result code
- **Inputs:** Result code to return to caller
- **Outputs/Return:** None
- **Side effects:** Sets `done = true` and `result` field
- **Calls:** None
- **Notes:** Simply sets internal state; `process_events()` will detect `done` flag on next call

### dialog::draw
- **Signature:** `void draw(void)`
- **Purpose:** Redraw entire dialog and all widgets to screen
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Renders frame, background, all widgets to SDL surface and updates screen
- **Calls:** `draw_widget()` (private) for each widget, theme functions
- **Notes:** Full redraw; see `draw_dirty_widgets()` for optimized partial redraw

### dialog::draw_dirty_widgets
- **Signature:** `void draw_dirty_widgets() const`
- **Purpose:** Redraw only widgets marked as `dirty == true` (optimization)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Renders dirty widgets only; updates screen
- **Calls:** `draw_widget()` (private) for dirty widgets
- **Notes:** Used automatically by dialog event handling; external code can call if manually altering widgets

### dialog::get_widget_by_id
- **Signature:** `widget *get_widget_by_id(short inID) const`
- **Purpose:** Find first widget with matching numeric ID
- **Inputs:** Widget ID to search for
- **Outputs/Return:** Pointer to widget or NULL if not found
- **Side effects:** None
- **Calls:** None
- **Notes:** Useful for external code to locate and manipulate specific widgets

### dialog::layout
- **Signature:** `void layout(void)`
- **Purpose:** Recompute widget positions and sizes using the configured layout manager
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates all widget `rect` fields; recalculates dialog dimensions
- **Calls:** `widget_placer::place()` (via `placer`), widget layout methods
- **Notes:** Exposed to allow toggling fullscreen; normally called automatically by `start()`

### dialog::set_widget_placer
- **Signature:** `void set_widget_placer(widget_placer *w)`
- **Purpose:** Set custom layout strategy (takes ownership)
- **Inputs:** Pointer to layout manager
- **Outputs/Return:** None
- **Side effects:** Replaces current placer; dialog takes ownership and will delete old one
- **Calls:** None
- **Notes:** Call before `start()` to customize layout; dialog becomes owner

### placeable::place
- **Signature:** `virtual void place(const SDL_Rect &r, placement_flags flags = kDefault) = 0`
- **Purpose:** Position and size this element within given rectangle
- **Inputs:** Target rectangle and alignment flags
- **Outputs/Return:** None
- **Side effects:** Updates internal position/size state
- **Calls:** None (overridden by subclasses)
- **Notes:** Called by parent layout manager; subclasses implement specific layout behavior

### table_placer::add / vertical_placer::add / horizontal_placer::add
- **Signature:** `void add(placeable *p, bool assume_ownership = false)` [varies slightly per class]
- **Purpose:** Add child element to layout
- **Inputs:** Child placeable; optional ownership flag
- **Outputs/Return:** None
- **Side effects:** Stores child reference; assumes ownership if flag is true
- **Calls:** None
- **Notes:** Placement details differ: table uses grid cells, vertical/horizontal use linear stacks

### widget_placer::assume_ownership
- **Signature:** `void assume_ownership(placeable *p)` [protected]
- **Purpose:** Mark child as owned (destructor will delete it)
- **Inputs:** Child placeable
- **Outputs/Return:** None
- **Side effects:** Adds to `m_owned` vector
- **Calls:** None
- **Notes:** Called by subclass `add()` methods; cleanup happens in `~widget_placer()`

## Control Flow Notes

**Modal dialogs** (`run()` path):
1. Save SDL state (cursor, unicode, key-repeat)
2. Call `start()` to initialize and draw
3. Loop: `process_events()` ΓåÆ if `done`, exit loop
4. Call `finish()` to clean up and restore state
5. Return result code

**Non-modal dialogs** (manual `start()`/`process_events()`/`finish()`):
1. Caller creates dialog, adds widgets, calls `start()`
2. Caller repeatedly calls `process_events()` in main game loop
3. When `done` flag is set (or external condition), caller calls `finish()`
4. Caller checks result code or accesses final widget state

**Widget focus management**:
- Only one widget has `active == true` at a time
- Keyboard events go to active widget
- Mouse events go to widget under cursor (`mouse_widget`)
- Tab/arrow keys rotate focus via `activate_next_widget()` / `activate_prev_widget()`

**Dirty widget optimization**:
- Widgets set `dirty = true` when they change
- `draw_dirty_widgets()` redraws only dirty widgets
- Used automatically on user interaction; external code can use for timer-driven or network-driven updates

## External Dependencies
- **SDL**: `SDL_Rect`, `SDL_Surface`, `SDL_Event` (graphics/input)
- **Boost**: `boost::function` (callback function objects)
- **Game engine (declared forward/elsewhere)**:
  - `widget` (derived from `placeable`; UI element base class)
  - `font_info` (font metrics and rendering)
  - `FileSpecifier` (file I/O abstraction)
  - `XML_ElementParser` (theme file parsing)
  - Sound IDs: `_snd_pattern_buffer`, `_snd_defender_hit`, etc.
