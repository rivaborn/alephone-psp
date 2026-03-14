# Source_Files/Misc/preferences_widgets_sdl.h

## File Purpose
Defines SDL-specific preference widget classes for environment/theme selection and crosshair display. Part of Aleph One's preferences dialog UI system, providing specialized widgets for file browsing and visual configuration elements.

## Core Responsibilities
- Theme/environment file discovery and enumeration (FindThemes)
- Environment item storage and hierarchy representation (env_item)
- List widget for browsing environment/theme files with selectable/non-selectable items (w_env_list)
- Selection button for choosing environment files with callback support (w_env_select)
- Graphical crosshair display widget (w_crosshair_display)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FindThemes` | class | FileFinder subclass that locates "theme2.mml" files in directories |
| `env_item` | struct | Represents a file/directory entry with indent level, name, and selectability flag |
| `w_env_list` | class | Template-instantiated list widget for displaying env_item entries with custom drawing |
| `w_env_select` | class | Selection button widget that opens a dialog to choose environment files |
| `w_crosshair_display` | class | Non-interactive display widget that renders a crosshair graphic (80├ù80px) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global (extern) | Search paths for data files, defined in shell_sdl.cpp |

## Key Functions / Methods

### FindThemes::found
- **Signature:** `bool found(FileSpecifier &file)`
- **Purpose:** Callback invoked by FileFinder for each file discovered. Identifies theme files and appends their parent directory to the destination vector.
- **Inputs:** FileSpecifier reference to the current file
- **Outputs/Return:** `false` (always; tells FileFinder to continue searching)
- **Side effects:** Appends parent directory path to `dest_vector` if filename is "theme2.mml"
- **Calls:** FileSpecifier::SplitPath(), vector::push_back()
- **Notes:** Extracts base path from file specifier; checks only the filename part (not full path) against "theme2.mml"

### w_env_list::w_env_list (constructor)
- **Signature:** `w_env_list(const vector<env_item> &items, const char *selection, dialog *d)`
- **Purpose:** Initializes the list widget with environment items and pre-selects an item matching the given path.
- **Inputs:** Item vector, selection path string, parent dialog pointer
- **Outputs/Return:** ΓÇö
- **Side effects:** Sets up list with 400├ù15 dimensions (from parent class); iterates items to find and select matching path
- **Calls:** w_list<env_item> constructor, FileSpecifier::GetPath(), set_selection()
- **Notes:** Performs string comparison on FileSpecifier paths to auto-select the item

### w_env_list::draw_item
- **Signature:** `void draw_item(vector<env_item>::const_iterator i, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- **Purpose:** Renders a single environment item to the list surface with indentation and color based on selectability.
- **Inputs:** Item iterator, target SDL surface, position (x, y), display width, selection state
- **Outputs/Return:** ΓÇö
- **Side effects:** Modifies SDL surface; calls set_drawing_clip_rectangle twice (before and after drawing)
- **Calls:** font->get_ascent(), get_theme_color(), set_drawing_clip_rectangle(), draw_text()
- **Notes:** Non-selectable items (directories) use LABEL_WIDGET colors; selectable items use ITEM_WIDGET colors with ACTIVE/DEFAULT states; indentation is x_offset = `indent * 8` pixels

### w_env_select::w_env_select (constructor)
- **Signature:** `w_env_select(const char *path, const char *m, Typecode t, dialog *d)`
- **Purpose:** Initializes environment selection button with file path, menu title, file type, and parent dialog reference.
- **Inputs:** Initial file path string, menu title string, file typecode, parent dialog pointer
- **Outputs/Return:** ΓÇö
- **Side effects:** Calls parent class constructor; calls set_arg(this) to register self as callback argument; calls set_path() to initialize display
- **Calls:** w_select_button constructor, set_arg(), set_path()
- **Notes:** Uses item_name and select_item_callback as callback functions for parent class

### w_env_select::set_path
- **Signature:** `void set_path(const char *p)`
- **Purpose:** Updates the currently selected file path and refreshes the button's display label.
- **Inputs:** File path string
- **Outputs/Return:** ΓÇö
- **Side effects:** Updates `item` (FileSpecifier), updates `item_name` array, calls set_selection() to refresh display
- **Calls:** FileSpecifier constructor/assignment, FileSpecifier::GetName(), set_selection()

### w_env_select::set_selection_made_callback
- **Signature:** `void set_selection_made_callback(selection_made_callback_t inCallback)`
- **Purpose:** Registers a callback function to be invoked when the user selects an item in the environment dialog.
- **Inputs:** Callback function pointer (w_env_select* ΓåÆ void)
- **Outputs/Return:** ΓÇö
- **Side effects:** Stores callback in `mCallback`
- **Calls:** (none)

### w_crosshair_display::draw
- **Signature:** `void draw(SDL_Surface *s) const`
- **Purpose:** Renders the crosshair display to the given SDL surface.
- **Inputs:** SDL surface pointer
- **Outputs/Return:** ΓÇö
- **Side effects:** Modifies SDL surface
- **Calls:** (implementation in .cpp file; not visible here)
- **Notes:** Defined elsewhere; size is fixed at 80├ù80 (kSize constant)

## Control Flow Notes
This file is part of preferences/settings dialog initialization. Likely flow:
1. **Init**: Theme discovery via FindThemes; environment list populated
2. **Interaction**: User clicks w_env_select button ΓåÆ select_item() opens dialog with w_env_list
3. **Selection**: User picks item ΓåÆ item_selected() closes dialog; optional callback fires
4. **Display**: w_crosshair_display renders continuously (is_dirty() always true)

Widgets integrate into SDL dialog system; they inherit from widget and w_list base classes.

## External Dependencies
- **Includes**: cseries.h, find_files.h, collection_definition.h, sdl_widgets.h, sdl_fonts.h, screen.h, screen_drawing.h, interface.h
- **Key external symbols**:
  - `FileFinder` (find_files.h) ΓÇö base class for file discovery
  - `FileSpecifier`, `DirectorySpecifier`, `Typecode` (file handling) ΓÇö file representation
  - `dialog` (sdl_dialogs.h via sdl_widgets.h) ΓÇö parent dialog context
  - `w_list<T>`, `w_select_button`, `widget` (sdl_widgets.h) ΓÇö parent widget classes
  - `font_info`, `get_theme_font()` (sdl_fonts.h) ΓÇö font management
  - `draw_text()`, `set_drawing_clip_rectangle()` (screen_drawing.h) ΓÇö rendering primitives
  - `get_theme_color()`, `get_theme_space()` (theme system) ΓÇö styling and layout
