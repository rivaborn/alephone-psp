# Source_Files/Files/FileHandler_SDL.cpp

## File Purpose
SDL-based implementation of platform-independent file handling for the Aleph One game engine. Provides abstractions for file I/O, resource management, file type detection, and modal dialogs for file selection, with transparent support for legacy Mac formats (AppleSingle, MacBinary).

## Core Responsibilities
- Wrap SDL_RWops file handles with error tracking and fork/offset handling for Mac compatibility
- Load and manage game resources (maps, sounds, shapes, physics data)
- Detect file types by reading magic bytes and version headers
- Search data directories for files using configurable search paths
- Provide directory traversal and file listing with custom widgets
- Implement modal file selection dialogs (open/save) with directory browsing
- Canonicalize and normalize file paths across platforms (Windows, Mac, Unix)
- Copy, rename, exchange, and delete files with error reporting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OpenedFile | class | Wraps SDL_RWops with position tracking, fork offsets, and error codes |
| LoadedResource | class | Manages malloc'd resource data with automatic cleanup |
| OpenedResourceFile | class | Manages resource fork file handles and context switching |
| FileSpecifier | class | Abstract file path with platform-specific path handling |
| DirectorySpecifier | typedef | Alias for FileSpecifier (platform-specific in headers) |
| dir_entry | struct | Directory listing entry with name, size, is_directory, is_volume flags |
| w_directory_browsing_list | class | Custom SDL widget for interactive directory navigation |
| w_file_list | class | Custom widget for displaying file listings |
| w_read_file_list | class | File list subclass with item-selected callback to parent dialog |
| w_write_file_list | class | File list with selection copying to filename textbox |
| w_file_name | class | Text entry widget that closes dialog on Return key |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | vector<DirectorySpecifier> | external (shell_sdl.cpp) | List of directories to search for data files |
| local_data_dir | DirectorySpecifier | external (shell_sdl.cpp) | Per-user temporary/config directory |
| preferences_dir | DirectorySpecifier | external (shell_sdl.cpp) | Per-user preferences directory |
| saved_games_dir | DirectorySpecifier | external (shell_sdl.cpp) | Per-user saved games directory |
| recordings_dir | DirectorySpecifier | external (shell_sdl.cpp) | Per-user film recordings directory |
| iDIRBROWSE_UP_BUTTON | enum | file-static | Widget ID for directory "up" button |
| iDIRBROWSE_DIR_NAME | enum | file-static | Widget ID for directory name display |
| iDIRBROWSE_BROWSER | enum | file-static | Widget ID for directory browser list |

## Key Functions / Methods

### OpenedFile::Open (FileSpecifier::Open overload, line ~315)
- Signature: `bool FileSpecifier::Open(OpenedFile& OFile, bool Writable)`
- Purpose: Open a data file via SDL_RWops, with transparent AppleSingle/MacBinary decompression on read
- Inputs: OFile (result), Writable flag
- Outputs/Return: bool (success); populates OFile.f, fork_offset, fork_length if Mac format detected
- Side effects: Opens file, may set game error via set_game_error()
- Calls: SDL_RWFromFile, open_fork_from_existing_path (Mac), is_applesingle, is_macbinary, SDL_RWseek
- Notes: On read, automatically detects and offsets into AppleSingle/MacBinary data fork; on write, returns immediately without format checks

### FileSpecifier::GetType (line ~377)
- Signature: `Typecode FileSpecifier::GetType()`
- Purpose: Infer file type by extension or by reading magic bytes (version, tags)
- Inputs: (implicit) file path
- Outputs/Return: Typecode enum (_typecode_sounds, _typecode_scenario, _typecode_physics, _typecode_shapes, _typecode_unknown)
- Side effects: Opens and seeks within file to detect format; closes file on return
- Calls: Open, SDL_ReadBE32, SDL_ReadBE16, SDL_RWseek, f.GetLength
- Notes: Fast-path for .dds, .jpg, .png extensions (returns unknown); checks version + tag fields at specific offsets for maps, sounds, shapes

### FileSpecifier::SetNameWithPath (line ~510)
- Signature: `bool FileSpecifier::SetNameWithPath(const char *NameWithPath)`
- Purpose: Locate a file in the search path by relative path (Unix-style /sep)
- Inputs: NameWithPath (e.g., "Shapes/Sprites.shp")
- Outputs/Return: bool (found); sets name to full path if found
- Side effects: Converts / to platform separator; iterates data_search_path
- Calls: Exists() for each search directory
- Notes: Returns ENOENT error if not found; always tries first match in search order

### FileSpecifier::ReadDialog (line ~699)
- Signature: `bool FileSpecifier::ReadDialog(Typecode type, const char *prompt)`
- Purpose: Present modal file-open dialog with directory browsing
- Inputs: type (determines starting directory), prompt (NULL uses defaults)
- Outputs/Return: bool; sets this->name on success
- Side effects: Blocks until dialog closed; redraws game window on exit; plays dialog sounds
- Calls: w_directory_browsing_list, dialog::run, update_game_window
- Notes: Saves/loads directory context by type; netscript type has special startup filename handling; callback updates directory display and up-button state

### FileSpecifier::WriteDialog (line ~800)
- Signature: `bool FileSpecifier::WriteDialog(Typecode type, const char *prompt, const char *default_name)`
- Purpose: Present modal file-save dialog with filename entry and existence confirmation
- Inputs: type (determines directory), prompt, default_name
- Outputs/Return: bool; sets this->name on success
- Side effects: Blocks; plays sounds; calls update_game_window; loops if user enters empty name or cancels overwrite confirmation
- Calls: ReadDirectory, confirm_save_choice, dialog::run, play_dialog_sound
- Notes: Implements retry-loop for validation (empty name, overwrite confirmation); closes dialog only on valid filename

### FileSpecifier::ReadDirectory (line ~606)
- Signature: `bool FileSpecifier::ReadDirectory(vector<dir_entry>& vec)`
- Purpose: List directory contents
- Inputs: (implicit) file path (must be directory)
- Outputs/Return: bool; populates vec with sorted dir_entry objects (excluding . and ..)
- Side effects: Allocates memory in vec
- Calls: Platform-specific: FindFirstFile/FindNextFile (Win32) or opendir/readdir (Unix)
- Notes: Sorts directories before files; uses FindFirstFile on Win32 with wildcard; uses stat for file type on Unix

### w_directory_browsing_list::item_selected (line ~557)
- Signature: `void w_directory_browsing_list::item_selected(void)`
- Purpose: Handle user selection of directory entry (navigate or select file)
- Inputs: (implicit) entries[selection]
- Outputs/Return: none
- Side effects: Modifies current_directory; may quit parent dialog (0 return code on file select) or refresh listing (on dir select)
- Calls: AddPart, refresh_entries, announce_directory_changed, parent_dialog->quit
- Notes: Directories trigger refresh + callback; files trigger dialog close

### confirm_save_choice (line ~867)
- Signature: `static bool confirm_save_choice(FileSpecifier& file)`
- Purpose: Confirm overwrite if file exists
- Inputs: file (must be set path)
- Outputs/Return: bool (true = proceed/not exists, false = cancelled)
- Side effects: Shows modal confirmation dialog
- Calls: Exists, dialog::run
- Notes: Dialog text shows filename via GetName; defaults to YES button

## Control Flow Notes
**Initialization**: File I/O is lazyΓÇöno explicit init() phase. Search paths and directories are populated by shell_sdl.cpp before file operations.

**Frame/Update**: No per-frame involvement; file operations are blocking modal calls (ReadDialog, WriteDialog).

**File Open Flow**: FileSpecifier::Open ΓåÆ SDL_RWFromFile or open_fork_from_existing_path (Mac) ΓåÆ transparent format detection (AppleSingle/MacBinary) ΓåÆ position/offset adjustment for forks.

**Type Detection**: GetType() attempts extension check first (fast path), falls back to opening file and reading magic bytes from known offsets (sounds version+tag, maps version+directory, shapes 32-entry header).

**Dialog Flow**: ReadDialog/WriteDialog create widget hierarchy (placer ΓåÆ widgets) ΓåÆ d.run() blocks until dialog result ΓåÆ cleanup and return.

## External Dependencies
- **SDL**: SDL_RWops, SDL_RWFromFile, SDL_RWread/write/seek/tell/close; SDL endian macros (SDL_ReadBE32, SDL_ReadBE16)
- **System**: stdio.h (FILE in non-Mac paths), stdlib.h, errno.h, limits.h, string, vector
- **Platform-specific**: 
  - Windows: windows.h, direct.h (mkdir), io.h (access), sys/stat.h (stat)
  - Unix: sys/stat.h, fcntl.h, dirent.h, unistd.h
- **Aleph One engine**: cseries.h, FileHandler.h, resource_manager.h (open_res_file, close_res_file, use_res_file, has_1_resource, get_1_resource), shell.h (search paths), interface.h (get_game_state, update_game_window), game_errors.h (set_game_error), tags.h (typecodes, FOUR_CHARS_TO_INT, tags like LINE_TAG, MONSTER_PHYSICS_TAG), sdl_dialogs.h (dialog, widget, placer classes), sdl_widgets.h (w_title, w_spacer, w_button, w_static_text, w_text_entry), SoundManager.h (play_dialog_sound)
- **Mac-specific**: mac_rwops.h (open_fork_from_existing_path), Functions for AppleSingle/MacBinary detection (is_applesingle, is_macbinary, extern from unknown source)
