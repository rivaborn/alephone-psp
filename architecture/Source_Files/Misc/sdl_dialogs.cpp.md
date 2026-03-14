# Source_Files/Misc/sdl_dialogs.cpp

## File Purpose

Implements the SDL-based dialog manager for the Aleph One game engine. Handles dialog lifecycle (initialization, display, event processing), theme loading from XML/MML files, and rendering of dialogs to the screen. Supports both software rendering and OpenGL-accelerated rendering.

## Core Responsibilities

- Initialize/shutdown dialog system and theme resources
- Load and parse theme definitions (colors, images, fonts, spacing) from XML
- Manage dialog modal display loop (start, run, finish, quit)
- Layout dialog and widgets using a placer-based system
- Handle input events (keyboard, mouse, fullscreen toggle)
- Render dialogs to SDL surface (or OpenGL when active)
- Manage dialog hierarchy (parent/child stacking)
- Activate/deactivate widgets and handle focus navigation (Tab, arrow keys)
- Support dirty-rect redrawing for incremental updates

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dialog` | class | Main dialog container; manages widgets, layout, event loop, rendering |
| `dialog_image_spec_type` | struct | Specifies image name and whether to scale |
| `theme_state` | struct | Color map, image specs, and loaded image surfaces for a widget state |
| `theme_widget` | struct | Font info and state-keyed theming (colors, images, spacing) for a widget type |
| `XML_ImageParser` | class | Parses `<image>` MML tags for widget states |
| `XML_DColorParser` | class | Parses `<color>` MML tags (RGB floats ΓåÆ SDL_Color) |
| `XML_DFontParser` | class | Parses `<font>` MML tags (ID, size, style, file paths) |
| `XML_DFrameParser` | class | Parses frame spacing (top/bottom/left/right) |
| `XML_ThemeWidgetParser` | class | Base for widget-type theme parsers (clears states on Start) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `top_dialog` | `dialog*` | global | Pointer to currently active (topmost) dialog; NULL if none |
| `dialog_surface` | `SDL_Surface*` | static | 640├ù480 16-bit offscreen buffer for dialog rendering |
| `default_image` | `SDL_Surface*` | static | 1├ù1 magenta surface used as fallback for missing images |
| `theme_resources` | `OpenedResourceFile` | static | Resource file handle for theme data |
| `dialog_theme` | `map<int, theme_widget>` | static | Widget-type ΓåÆ theming info; indexed by enum (BUTTON_WIDGET, etc.) |
| `foundLabelOutlineColor` | `bool` | static | Tracks if label outline color was parsed |
| Various static `XML_*Parser` instances | `XML_*Parser` | static | ~40+ parser instances for different widget types/states |
| `dialog::sKeyRepeatActive` | `bool` | static class member | SDL key-repeat state (not stored by SDL itself) |

## Key Functions / Methods

### initialize_dialogs
- **Signature:** `void initialize_dialogs(FileSpecifier &theme)`
- **Purpose:** Set up the dialog system at startup; allocate surfaces and load theme.
- **Inputs:** `theme` ΓÇö file specifier for theme to load
- **Outputs/Return:** None
- **Side effects:** Allocates `dialog_surface` and `default_image`; attempts to load theme; registers `shutdown_dialogs()` with `atexit()`
- **Calls:** `SDL_CreateRGBSurface()`, `SDL_FillRect()`, `SDL_SetColorKey()`, `load_theme()`, `get_default_theme_spec()`, `write_preferences()`, `atexit()`
- **Notes:** Falls back to default theme if specified theme fails to load

### load_theme
- **Signature:** `bool load_theme(FileSpecifier &theme)` (definition not shown, but called)
- **Purpose:** Load theme colors, images, fonts, and spacing from a theme file via XML parsing
- **Notes:** Populates `dialog_theme` map; implemented elsewhere

### unload_theme
- **Signature:** `static void unload_theme(void)`
- **Purpose:** Free theme resources (surfaces, fonts, etc.)
- **Notes:** Called by `shutdown_dialogs()`

### dialog::add
- **Signature:** `void dialog::add(widget *w)`
- **Purpose:** Add a widget to the dialog's widget list and set its owning dialog.
- **Inputs:** `w` ΓÇö widget pointer (takes ownership)
- **Outputs/Return:** None
- **Side effects:** Appends to `widgets` vector; calls `w->set_owning_dialog(this)`

### dialog::layout
- **Signature:** `void dialog::layout()`
- **Purpose:** Position and size the dialog and all widgets; center on screen.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calculates `rect` (dialog position/size); calls `placer->place()` for widget layout; updates `layout_for_fullscreen`
- **Calls:** `placer->min_width()`, `placer->min_height()`, `placer->place()`, `get_theme_space()`, `SDL_GetVideoSurface()`
- **Notes:** Handles fullscreen mode changes; called at start and if fullscreen toggle detected

### dialog::draw
- **Signature:** `void dialog::draw(void)`
- **Purpose:** Clear and redraw entire dialog and all visible widgets.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `dialog_surface`; draws frame (image-based or solid outline); draws all widgets; blits to screen via `update()`
- **Calls:** `layout()`, `SDL_FillRect()`, `draw_frame_image()`, `use_theme_images()`, `get_theme_color()`, `draw_rectangle()`, `draw_widget()`, `update()`

### dialog::draw_widget
- **Signature:** `void dialog::draw_widget(widget *w, bool do_update) const`
- **Purpose:** Clear widget's area and redraw it.
- **Inputs:** `w` ΓÇö widget to draw; `do_update` ΓÇö whether to blit to screen immediately
- **Outputs/Return:** None
- **Side effects:** Fills widget rect with background color, calls `w->draw()`, clears dirty flag, optionally calls `update()`

### dialog::update
- **Signature:** `void dialog::update(SDL_Rect r) const`
- **Purpose:** Blit dialog surface region to screen (software or OpenGL).
- **Inputs:** `r` ΓÇö region within dialog surface to update
- **Outputs/Return:** None
- **Side effects:** If OpenGL active, converts to 32-bit RGBA and uses glDrawPixels; otherwise uses SDL_BlitSurface
- **Calls:** `OGL_IsActive()`, `glPushAttrib()`, `glMatrixMode()`, `gluOrtho2D()`, `glDrawPixels()`, `SDL_FreeSurface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`
- **Notes:** OpenGL path handles coordinate and endianness conversion; includes extensive GL state setup/restore

### dialog::event
- **Signature:** `void dialog::event(SDL_Event &e)`
- **Purpose:** Dispatch input events to widgets or handle dialog-level commands.
- **Inputs:** `e` ΓÇö SDL event
- **Outputs/Return:** None
- **Side effects:** May activate/deactivate widgets, trigger widget callbacks, modify dialog state (quit, fullscreen), call `draw_dirty_widgets()`
- **Calls:** `toggle_fullscreen()`, `draw()`, `active_widget->event()`, `find_widget()`, `mouse_widget->event()`, `activate_next_widget()`, `activate_prev_widget()`, `quit()`, `dump_screen()`, `draw_dirty_widgets()`, `play_dialog_sound()`
- **Notes:** Handles KEYDOWN (ESC, arrows, Tab, Enter, F9), MOUSEMOTION, MOUSEBUTTONDOWN/UP, ACTIVEEVENT, QUIT; first pass to active widget before own handling

### dialog::run
- **Signature:** `int dialog::run(bool intro_exit_sounds)`
- **Purpose:** Display dialog modally and block until user closes it; return result code.
- **Inputs:** `intro_exit_sounds` ΓÇö whether to play sounds on open/close
- **Outputs/Return:** Result code (0=OK, -1=cancel)
- **Side effects:** Calls `start()`, runs event loop with `process_events()` and optional `processing_function()`, calls `finish()`, may redraw previous dialog if nested
- **Calls:** `start()`, `process_events()`, `processing_function()`, `global_idle_proc()`, `SDL_Delay()`, `finish()`

### dialog::start
- **Signature:** `void dialog::start(bool play_sound)`
- **Purpose:** Prepare dialog for display: layout, load frame images, show cursor, enable keyboard input.
- **Inputs:** `play_sound` ΓÇö whether to play intro sound
- **Outputs/Return:** None
- **Side effects:** Sets `top_dialog`, clears `dialog_surface`, loads 8 frame images, draws dialog, saves/sets Unicode and key-repeat state, plays sound, initializes event loop state
- **Calls:** `layout()`, `get_theme_image()`, `draw()`, `SDL_ShowCursor()`, `play_dialog_sound()`, `SDL_EnableUNICODE()`, `SDL_EnableKeyRepeat()`
- **Notes:** Can be nested (saves `parent_dialog`); handles OpenGL-specific GL clear if applicable

### dialog::finish
- **Signature:** `int dialog::finish(bool play_sound)`
- **Purpose:** Clean up after dialog: restore state, free resources, erase from screen.
- **Inputs:** `play_sound` ΓÇö whether to play exit sound
- **Outputs/Return:** Result code
- **Side effects:** Restores Unicode/key-repeat state, plays sound, hides cursor, clears dialog surface, frees frame images, restores parent dialog and redraws it if present
- **Calls:** `SDL_EnableUNICODE()`, `SDL_EnableKeyRepeat()`, `play_dialog_sound()`, `SDL_ShowCursor()`, `SDL_FillRect()`, `SDL_FreeSurface()`, `clear_screen()`, `glClear()`, `top_dialog->draw()`

### dialog::process_events
- **Signature:** `bool dialog::process_events()`
- **Purpose:** Non-blocking drain of pending SDL events; dispatch to `event()`.
- **Inputs:** None
- **Outputs/Return:** `done` flag (true if dialog should close)
- **Side effects:** Polls SDL event queue until empty; calls `event()` for each
- **Calls:** `psp_mouse->update()`, `SDL_PollEvent()`, `event()`
- **Notes:** PSP mouse update for platform-specific input

### dialog::activate_next_widget / dialog::activate_prev_widget
- **Signature:** `void dialog::activate_next_widget(void)` / `void dialog::activate_prev_widget(void)`
- **Purpose:** Move input focus to next/previous selectable, visible widget.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `deactivate_currently_active_widget()` and `activate_widget()` to change focus
- **Notes:** Wraps around; skips labels and their associated widgets to avoid dual selection; contains infinite-loop BUG comments

### dialog::activate_widget
- **Signature:** `void dialog::activate_widget(widget *w, bool draw)` or `void dialog::activate_widget(size_t num, bool draw)`
- **Purpose:** Give input focus to a specific widget; deactivate previous and update labels.
- **Inputs:** `w` or `num` ΓÇö widget pointer or index; `draw` ΓÇö whether to redraw
- **Outputs/Return:** None
- **Side effects:** Sets `active_widget` and `active_widget_num`; handles associated labels; optionally redraws
- **Calls:** `find_widget()` (pointer version), `deactivate_currently_active_widget()`, `draw_widget()`
- **Notes:** Handles label-widget associations (label must be immediately before/after widget)

### dialog::find_widget
- **Signature:** `int dialog::find_widget(int x, int y)`
- **Purpose:** Hit-test screen coordinates; return index of widget under cursor.
- **Inputs:** `x`, `y` ΓÇö video surface coordinates
- **Outputs/Return:** Widget index (0+) or -1 if none
- **Notes:** Transforms to dialog coordinates; checks visibility

### dialog::draw_dirty_widgets
- **Signature:** `void dialog::draw_dirty_widgets() const`
- **Purpose:** Efficiently redraw only widgets marked as needing update.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates widgets; calls `draw_widget()` for dirty, visible ones
- **Calls:** `widgets[i]->is_dirty()`, `widgets[i]->visible()`, `draw_widget()`
- **Notes:** Only processes if this dialog is `top_dialog`; called automatically after event processing

### XML_ImageParser, XML_DColorParser, XML_DFontParser classes
- **Purpose:** Parse MML theme elements; populate `dialog_theme` map during theme load
- **Methods:** `Start()` (reset state), `HandleAttribute()` (process tag attributes), `AttributesDone()` (validate and commit)
- **Notes:** ~40+ static instances created for different widget types and states

## Control Flow Notes

1. **Startup:** `initialize_dialogs()` allocates surfaces, loads theme from prefs or default
2. **Theme Load:** XML parsers incrementally populate `dialog_theme` as MML is parsed
3. **Dialog Display:** `dialog::run()` calls `start()` (layout, draw, setup), then loops calling `process_events()` until `done=true`, then `finish()` (cleanup, erase, restore)
4. **Input Handling:** Events dispatched via `event()` ΓåÆ active widget first, then dialog-level (ESC quits, arrows navigate, Enter clicks, Tab cycles)
5. **Rendering:** Dialogs render to `dialog_surface`, then blit to screen (SDL) or OpenGL framebuffer
6. **Nesting:** Each dialog saves `parent_dialog` and restores/redraws it on close
7. **Dirty Tracking:** Widgets mark `dirty=true` when they change; `draw_dirty_widgets()` redraws only those on idle

## External Dependencies

- **SDL:** `SDL_Surface`, `SDL_Event`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FillRect()`, `SDL_ShowCursor()`, `SDL_EnableUNICODE()`, `SDL_EnableKeyRepeat()`, key constants (SDLK_*)
- **OpenGL (optional):** `glPushAttrib()`, `glMatrixMode()`, `gluOrtho2D()`, `glDrawPixels()`, `glClear()`, `OGL_IsActive()`, `OGL_Setup.h`
- **Engine Integration:** `shell.h` (global_idle_proc, get_screen_mode), `screen.h` (toggle_fullscreen, clear_screen, dump_screen), `screen_drawing.h` (draw_rectangle), `sdl_widgets.h` (widget, w_label classes), `sdl_fonts.h` (font_info), `SoundManager.h` (play_dialog_sound)
- **XML Parsing:** `XML_Loader_SDL.h`, `XML_ParseTreeRoot.h`, `XML_ElementParser` base class
- **File Handling:** `FileSpecifier`, `OpenedResourceFile` (theme resources), `preferences.h` (environment_preferences)
