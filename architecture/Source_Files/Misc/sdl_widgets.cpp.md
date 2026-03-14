# Source_Files/Misc/sdl_widgets.cpp

## File Purpose
Implements a comprehensive SDL-based UI widget library for dialog boxes and UI elements in the Aleph One game engine. Provides specialized widgets for text rendering, buttons, selections, lists, and metaserver integration (network games/players), with support for dynamic state management, callbacks, and theming.

## Core Responsibilities
- Base widget infrastructure with lifecycle, state (enabled/disabled), focus, and dirty-marking
- Text rendering widgets (static text, clickable labels)
- Interactive input widgets (buttons, selections, text entry, sliders)
- Scrollable list widgets with selection and keyboard navigation
- Specialized network/metaserver widgets (game lists, player lists, chat)
- Tab widget for tabbed interfaces
- File chooser and resource-based image display
- Event dispatching (mouse, keyboard) and callback invocation
- Integration with theming system for colors/images/fonts

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| widget | class | Base class for all UI widgets; manages position, state, focus, and dirtyness |
| w_static_text | class | Displays non-interactive text; owns its own string copy |
| w_label | class | Clickable label that can forward clicks to an associated widget |
| w_button_base | class | Button with multiple visual states (default, active, disabled, pressed) |
| w_tab | class | Tabbed interface; manages multiple labeled tabs with a placer |
| w_select | class | Cyclic selector widget; cycles through label options on click or arrow keys |
| w_select_button | class | Button-like widget displaying a selection string |
| w_select_popup | class | Popup selector; opens a dialog with a string list on click |
| w_slider | class | Numeric slider with draggable thumb; range-based selection |
| w_list_base | class | Base scrollable list with keyboard/mouse navigation and scroll thumb |
| w_list<T> | template class | Typed list template for vector-based item collections |
| w_games_in_room | class | Specialized list displaying network GameListEntry with game info and status colors |
| w_players_in_room | class | Specialized list displaying MetaserverPlayerInfo with player colors and team swatches |
| w_colorful_chat | class | Chat widget displaying colored chat entries with sender names and message types |
| w_file_chooser | class | File selection button; displays filename and opens native file dialog on click |
| ColoredChatEntry | struct | Chat message container with type (chat/private/server/local), sender, message, and color |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sNoValidOptionsString | static const char* | file-static | Fallback string displayed when w_select has no valid options |
| sFileChooserInvalidFileString | static const char* | file-static | Fallback string displayed when w_file_chooser has no valid selection |

## Key Functions / Methods

### widget::widget (constructors)
- Signature: `widget()`, `widget(int theme_widget)`
- Purpose: Initialize base widget state (position, flags, theme)
- Inputs: Optional theme widget type ID
- Outputs/Return: None
- Side effects: Initializes rect, active, dirty, enabled, font, identifier, owning_dialog to defaults
- Calls: `get_theme_font()` (conditionally)
- Notes: Default ctor initializes empty state; theme ctor loads font from theme system

### widget::set_enabled
- Signature: `void set_enabled(bool inEnabled)`
- Purpose: Enable or disable widget; updates visual state and focus
- Inputs: inEnabled ΓÇö desired enabled state
- Outputs/Return: None
- Side effects: Sets dirty flag; calls `owning_dialog->activate_next_widget()` if widget had focus while being disabled; propagates to associated_label
- Calls: `owning_dialog->activate_next_widget()` (if active && !enabled)
- Notes: Disabling a focused widget automatically transfers focus; associated labels are also disabled

### widget::place
- Signature: `void place(const SDL_Rect &r, placement_flags flags)`
- Purpose: Position and size widget according to layout flags
- Inputs: r ΓÇö available rectangle; flags ΓÇö alignment/fill directives
- Outputs/Return: None
- Side effects: Updates rect position and dimensions
- Calls: None
- Notes: Supports kFill (expand to fill), kAlignLeft/Center/Right; respects saved_min_width

### w_static_text::w_static_text (constructor)
- Signature: `w_static_text(const char *t, int _theme_type = MESSAGE_WIDGET)`
- Purpose: Create static text widget with owned string copy
- Inputs: t ΓÇö text string; _theme_type ΓÇö widget theme type
- Outputs/Return: None
- Side effects: Allocates text buffer via strdup(); calculates and stores min width/height
- Calls: `font->text_width()`, `font->get_line_height()`
- Notes: Constructor calculates dimensions once; destructor frees text buffer

### w_static_text::set_text
- Signature: `void set_text(const char* t)`
- Purpose: Update text after widget creation
- Inputs: t ΓÇö new text string
- Outputs/Return: None
- Side effects: Frees old text buffer; allocates new one; marks dirty for redraw
- Calls: `free()`, `strdup()`
- Notes: Allows dynamic text changes (e.g., status messages)

### w_button_base::draw
- Signature: `void draw(SDL_Surface *s) const`
- Purpose: Render button with current state (default/active/disabled/pressed)
- Inputs: s ΓÇö target SDL surface
- Outputs/Return: None
- Side effects: Blits button images or fills rectangles; draws text
- Calls: `get_theme_image()`, `SDL_BlitSurface()`, `get_theme_color()`, `draw_text()` (multiple)
- Notes: Supports both image-based (tiled left/center/right) and color-based rendering

### w_button_base::mouse_down, mouse_up, mouse_move
- Signature: `void mouse_down(int, int)`, `void mouse_up(int x, int y)`, `void mouse_move(int x, int y)`
- Purpose: Handle button press/release and drag-tracking
- Inputs: x, y ΓÇö cursor position relative to widget
- Outputs/Return: None
- Side effects: Sets down/pressed flags; marks dirty; invokes callback on mouse_up if within bounds
- Calls: `proc()` (callback), `get_owning_dialog()->draw_dirty_widgets()`
- Notes: mouse_move tracks whether cursor is still over button while dragging; click fired only on release within bounds

### w_select::draw
- Signature: `void draw(SDL_Surface *s) const`
- Purpose: Render current selection label
- Inputs: s ΓÇö target SDL surface
- Outputs/Return: None
- Side effects: None
- Calls: `draw_text()`, `get_theme_color()`
- Notes: Falls back to "(no valid options)" string if num_labels == 0

### w_select::click
- Signature: `void click(int /*x*/, int /*y*/)`
- Purpose: Cycle to next selection on click
- Inputs: x, y ΓÇö unused (click position)
- Outputs/Return: None
- Side effects: Increments selection (wraps to 0); marks dirty; invokes selection_changed_callback
- Calls: `selection_changed()`
- Notes: Wraps selection at num_labels boundary; only active if enabled

### w_select::event
- Signature: `void event(SDL_Event &e)`
- Purpose: Handle left/right arrow key navigation
- Inputs: e ΓÇö SDL keyboard event
- Outputs/Return: None (consumes event by setting e.type = SDL_NOEVENT)
- Side effects: Updates selection; marks dirty; consumes event to prevent propagation
- Calls: `selection_changed()`
- Notes: Left/right arrows cycle selection; event is consumed to prevent default dialog handling

### w_list_base::draw
- Signature: `void draw(SDL_Surface *s) const`
- Purpose: Render list frame, items, and scroll thumb
- Inputs: s ΓÇö target SDL surface
- Outputs/Return: None
- Side effects: None
- Calls: `draw_items()` (virtual), `draw_rectangle()`, `SDL_BlitSurface()` (for frame/thumb images)
- Notes: Delegates item rendering to subclass; draws frame around entire list and scroll thumb

### w_list_base::set_selection
- Signature: `void set_selection(size_t s)`
- Purpose: Change selected item and ensure it is visible
- Inputs: s ΓÇö new selection index (bounds-checked with assertion)
- Outputs/Return: None
- Side effects: Updates selection; marks dirty; adjusts top_item to keep selection visible via set_top_item()
- Calls: `set_top_item()`
- Notes: Asserts that selection is in valid range [0, num_items-1]

### w_list_base::new_items
- Signature: `void new_items(void)`
- Purpose: Recalculate list layout (thumb height, visible items) after item collection changes
- Inputs: None (uses num_items, shown_items)
- Outputs/Return: None
- Side effects: Recalculates thumb height and creates dynamic thumb images; tries to preserve selection/top_item if possible
- Calls: `get_theme_image()`, `SDL_FreeSurface()` (for old thumb images), `set_selection()`, `set_top_item()`
- Notes: Handles edge cases (empty list, thumb too small); preserves user's scroll position if selection still valid

### w_games_in_room::draw_item
- Signature: `void draw_item(const GameListMessage::GameListEntry& item, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const`
- Purpose: Render a single game entry in the metaserver game list
- Inputs: item ΓÇö game entry; s ΓÇö target surface; x, y ΓÇö position; width ΓÇö available width; selected ΓÇö if item is selected
- Outputs/Return: None
- Side effects: Draws multi-line game info (name, map, player count, time limit, teams)
- Calls: `SDL_FillRect()`, `font->draw_styled_text()`, `draw_text()`, `set_drawing_clip_rectangle()`
- Notes: Color-codes games by state (compatible/incompatible/running); shows remaining time or ping info

### w_players_in_room::draw_item
- Signature: `void draw_item(const MetaserverPlayerInfo& item, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const`
- Purpose: Render a single player entry with colored swatches and name
- Inputs: item ΓÇö player info; s ΓÇö target surface; x, y ΓÇö position; width ΓÇö available width; selected ΓÇö if item is selected
- Outputs/Return: None
- Side effects: Draws player color swatch, team color swatch, and name with optional styling
- Calls: `darken()`, `lighten()`, `SDL_MapRGB()`, `SDL_FillRect()`, `font->draw_styled_text()`
- Notes: Darkens colors for away players; lightens colors for targeted player; supports team colors

### w_colorful_chat::append_entry
- Signature: `void append_entry(const ColoredChatEntry& e)`
- Purpose: Add and display a chat entry; wraps long messages across lines
- Inputs: e ΓÇö colored chat entry to append
- Outputs/Return: None
- Side effects: Breaks message into multiple entries if too wide; updates num_items; calls new_items(); tries to preserve scroll position
- Calls: `font->trunc_styled_text()`, `font->style_at()`, `new_items()`, `set_top_item()`, `append_entry()` (recursive)
- Notes: Recursively appends wrapped portions to maintain formatting; preserves auto-scroll to bottom unless user scrolled up

### w_colorful_chat::draw_item
- Signature: `void draw_item(vector<ColoredChatEntry>::const_iterator it, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- Purpose: Render a single chat entry line (with sender name for chat/private messages, or full-width for system messages)
- Inputs: it ΓÇö iterator to entry; s ΓÇö target surface; x, y ΓÇö position; width ΓÇö available width; selected ΓÇö if item is selected
- Outputs/Return: None
- Side effects: Draws sender name swatch, taper effect, and message text with type-specific colors/backgrounds
- Calls: `SDL_MapRGB()`, `SDL_FillRect()`, `font->draw_styled_text()`, `set_drawing_clip_rectangle()`
- Notes: Chat messages have a colored name bar with a taper; private messages add a red underbar; system/local messages are gray with bar

### w_file_chooser::click
- Signature: `void click(int, int)`
- Purpose: Open native file selection dialog
- Inputs: x, y ΓÇö unused
- Outputs/Return: None
- Side effects: Calls `file.ReadDialog()` to open file picker; updates display if file selected; invokes m_callback
- Calls: `FileSpecifier::ReadDialog()`, `update_filename()`, `m_callback()` (if set)
- Notes: Only active if widget is enabled; updates displayed filename or fallback string based on selection result

### w_file_chooser::update_filename
- Signature: `void update_filename()`
- Purpose: Refresh displayed filename from FileSpecifier
- Inputs: None
- Outputs/Return: None
- Side effects: Gets filename from file specifier and updates widget display; shows fallback string if file doesn't exist
- Calls: `FileSpecifier::Exists()`, `FileSpecifier::GetName()`, `set_selection()`
- Notes: Called after file dialog or programmatic file changes

## Control Flow Notes
- **Event Loop**: Dialog handles SDL events (mouse/keyboard) and dispatches to active/targeted widget
- **Input Flow**: widget::event/click/mouse_* ΓåÆ callback invocation (if set) ΓåÆ dialog::draw_dirty_widgets() (if dirty flag set)
- **List Scroll**: Wheel events (WHEELUP/WHEELDOWN) adjust top_item via set_top_item(); keyboard arrows (UP/DOWN) adjust selection
- **Chat Append**: New entries are recursively split if exceeding width; scroll position is preserved unless user scrolled up
- **Dirty Marking**: Widgets set dirty flag on state changes; dialog redraws only dirty widgets on request (not every frame)

## External Dependencies
- **SDL.h** ΓÇö graphics rendering (SDL_Surface, SDL_Rect, SDL_BlitSurface, SDL_FillRect, SDL_MapRGB, SDL_Delay)
- **cseries.h** ΓÇö common game engine utilities and macros (PIN, FOUR_CHARS_TO_INT)
- **sdl_dialogs.h** ΓÇö dialog class, placeable interface, theme constants (BUTTON_WIDGET, LABEL_WIDGET, etc.)
- **sdl_fonts.h** ΓÇö font_info interface for text measurement and rendering
- **screen_drawing.h** ΓÇö draw_text(), draw_rectangle(), set_drawing_clip_rectangle() utilities
- **resource_manager.h** ΓÇö LoadedResource, get_resource() for loading PICT resources
- **shape_descriptors.h** ΓÇö shape descriptor constants (unused in this file's content)
- **TextStrings.h** ΓÇö stringset support for dynamic label loading
- **mouse.h** ΓÇö mouse button constants (NUM_SDL_MOUSE_BUTTONS, SDLK_BASE_MOUSE_BUTTON)
- **map.h** ΓÇö entry_point struct for w_levels widget
- **tags.h** ΓÇö Typecode enum for w_file_chooser
- **FileHandler.h** ΓÇö FileSpecifier for file selection
- **metaserver_messages.h** ΓÇö GameListMessage, GameListEntry for w_games_in_room
- **network.h** ΓÇö prospective_joiner_info, MetaserverPlayerInfo for network lists
- **binders.h** ΓÇö boost::bind/boost::function for callback support
