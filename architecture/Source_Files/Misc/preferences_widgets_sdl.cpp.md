# Source_Files/Misc/preferences_widgets_sdl.cpp

## File Purpose
Implements SDL-specific preference UI widgets for the Aleph One game engine, providing file/environment selection dialogs and crosshair preview rendering. Extracted from the main preferences module to enable code sharing across platforms.

## Core Responsibilities
- Implement environment file selection dialog (w_env_select) for themes, maps, physics, shapes, and sounds
- Build and display hierarchical file lists with indentation and selective item availability
- Manage directory traversal across multiple search paths for asset discovery
- Implement crosshair display widget (w_crosshair_display) for UI preview rendering
- Handle theme file discovery and structured presentation to users

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `w_env_select` | class (widget subclass) | Preference button widget that opens file selection dialog; stores selected FileSpecifier and invokes optional callback |
| `w_env_list` | class (w_list<env_item> template) | List widget for displaying environment items with indent levels and selectability flags |
| `env_item` | struct | Single file/directory entry with FileSpecifier, name, indent level, and selectable flag |
| `FindThemes` | class (FileFinder subclass) | Custom file finder that filters for "theme2.mml" files and returns parent directory paths |
| `w_crosshair_display` | class (widget subclass) | Non-selectable widget that renders crosshair preview to an internal SDL surface |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global | External vector of directories to search for asset files; populated by shell_sdl.cpp |
| `local_data_dir` | `DirectorySpecifier` | global (macOS only) | External macOS-specific local data directory |

## Key Functions / Methods

### w_env_select::select_item_callback
- Signature: `static void select_item_callback(void* arg)`
- Purpose: Static callback wrapper for button click events, delegates to instance method
- Inputs: `void* arg` (cast to `w_env_select*`)
- Outputs/Return: void
- Side effects: Invokes select_item on the object instance
- Calls: `obj->select_item(obj->parent)`
- Notes: Used as a function pointer callback; contains type-unsafe void* cast

### w_env_select::select_item
- Signature: `void w_env_select::select_item(dialog *parent)`
- Purpose: Display file selection dialog allowing user to choose environment files (themes, maps, physics, shapes, sounds) from multiple search paths
- Inputs: `parent` dialog pointer (also accesses `type`, `menu_title`, `item` via member variables)
- Outputs/Return: void (user selection stored via `set_path()` and optional callback invoked)
- Side effects: 
  - Searches file system using FindThemes or FindAllFiles across all data_search_path directories
  - Clears screen via `clear_screen()`
  - Creates and runs modal dialog
  - Updates `item` member variable if user accepts
  - Invokes `mCallback` callback if set and user accepts
- Calls: 
  - `FindThemes()` or `FindAllFiles()` constructors
  - `finder.Find(dir, type/WILDCARD_TYPE)` (iteration)
  - `FileSpecifier::SplitPath()`, `GetName()`, `GetPath()`
  - `dialog::activate_widget()`, `set_widget_placer()`, `run()`
  - `clear_screen()`, `set_path()`, `mCallback()`
- Notes:
  - Branches on `type == _typecode_theme` to use special FindThemes finder vs. generic FindAllFiles
  - Builds hierarchical tree: directories become unselectable header items; files are indented beneath them
  - On macOS, top-level directories (where `GetName()` returns empty string) skip header insertion and use indent level 0
  - Dialog accepted only if user clicks item and presses OK (confirmed by `d.run() == 0`)

### w_crosshair_display::w_crosshair_display
- Signature: `w_crosshair_display()`
- Purpose: Construct the crosshair display widget and allocate internal SDL rendering surface
- Inputs: none
- Outputs/Return: object instance
- Side effects:
  - Allocates SDL_Surface (16-bit RGB, 80x80 pixels) with specific color masks
  - Initializes `rect` dimensions and minimum layout constraints to `kSize` (80)
- Calls: `SDL_CreateRGBSurface(SDL_SWSURFACE, kSize, kSize, 16, 0x7c00, 0x03e0, 0x001f, 0)`
- Notes: Fixed size of 80├ù80; surface pointer stored in member `surface`; mask values define 5-5-5-1 RGB format

### w_crosshair_display::~w_crosshair_display
- Signature: `~w_crosshair_display()`
- Purpose: Destroy widget and release SDL surface resource
- Inputs: none
- Outputs/Return: object destroyed
- Side effects: Frees SDL surface; nulls `surface` pointer
- Calls: `SDL_FreeSurface(surface)`
- Notes: Resource cleanup pattern; implicit null-check safety

### w_crosshair_display::draw
- Signature: `void w_crosshair_display::draw(SDL_Surface *s) const`
- Purpose: Render crosshair preview to destination surface by drawing to internal buffer and blitting
- Inputs: `SDL_Surface *s` (destination surface to blit to)
- Outputs/Return: void
- Side effects:
  - Fills internal surface with theme background color
  - Draws themed frame rectangle on internal surface
  - Temporarily enables crosshairs and invokes crosshair render system
  - Restores crosshair active state to prior value
  - Blits internal surface to destination surface at `rect` position
- Calls: 
  - `SDL_FillRect(surface, 0, color)`
  - `draw_rectangle(surface, &r, color)`
  - `get_theme_color()` (twice: DIALOG_FRAME BACKGROUND_COLOR, DIALOG_FRAME FRAME_COLOR)
  - `Crosshairs_IsActive()`, `Crosshairs_SetActive()`, `Crosshairs_Render(surface)`
  - `SDL_BlitSurface(surface, 0, s, &rect)`
- Notes:
  - const method; does not modify object state
  - Temporarily modifies global crosshair active state; restores via saved state variable
  - Integrates with external crosshair rendering system and theme color system

## Control Flow Notes
This file is **event-driven** rather than frame-based. The environment selection is triggered by user interaction (button click) and displays a modal dialog. The crosshair display is a **reactive widget** rendered on-demand when its parent dialog is active. No initialization, shutdown, or per-frame update logic is present.

## External Dependencies
- **SDL Graphics**: `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface`, `SDL_FreeSurface`, `SDL_FillRect`, `SDL_BlitSurface`
- **Aleph One Framework**: `dialog`, `widget`, `w_select_button`, `w_list<T>`, `vertical_placer`, `FileSpecifier`, `DirectorySpecifier`, `Typecode` (defined elsewhere)
- **File Finding**: `FileFinder`, `FindAllFiles` (from find_files.h), `FindThemes` (defined in header)
- **UI Rendering**: `w_title`, `w_spacer`, `w_button`, `w_env_list`, theme color system (`get_theme_color`), drawing functions (`draw_rectangle`, `set_drawing_clip_rectangle`, `draw_text`) (from screen_drawing.h)
- **Crosshair System**: `Crosshairs_IsActive()`, `Crosshairs_SetActive()`, `Crosshairs_Render()` (from Crosshairs.h)
- **Globals**: `data_search_path` (from shell_sdl.cpp), `local_data_dir` (macOS only)
