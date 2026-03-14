# Source_Files/Misc/progress.h

## File Purpose
Header file that defines the interface for displaying progress dialogs and progress bars during network operations (map/physics distribution), loading, and router configuration. Part of the Aleph One game engine (Marathon engine), providing user feedback during long-running operations.

## Core Responsibilities
- Define message ID constants for various progress states (network sync, physics sync, map loading, router management)
- Provide lifecycle management for progress dialogs (open, close, update message)
- Expose progress bar drawing and reset functionality
- Support conditional progress bar display (with `show_progress_bar` flag)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Anonymous enum | enum | Message IDs for progress dialog states; base resource ID `strPROGRESS_MESSAGES` (143) used to index localized strings |

## Global / File-Static State
None.

## Key Functions / Methods

### open_progress_dialog
- Signature: `void open_progress_dialog(size_t message_id, bool show_progress_bar = false)`
- Purpose: Display a progress dialog with optional progress bar
- Inputs: `message_id` (index into resource 143 for localized message), `show_progress_bar` (controls visibility of bar)
- Outputs/Return: None
- Side effects: Creates UI dialog; likely allocates/displays window

### close_progress_dialog
- Signature: `void close_progress_dialog(void)`
- Purpose: Dismiss the active progress dialog
- Inputs: None
- Outputs/Return: None
- Side effects: Destroys UI dialog; likely deallocates window resources

### set_progress_dialog_message
- Signature: `void set_progress_dialog_message(size_t message_id)`
- Purpose: Update the message displayed in an open progress dialog
- Inputs: `message_id` (index into resource 143)
- Outputs/Return: None
- Side effects: Updates UI text

### draw_progress_bar / reset_progress_bar
- Purpose: Draw progress bar with byte counts or reset progress state
- `draw_progress_bar(size_t sent, size_t total)` ΓÇö renders bar; used for network/file operations
- `reset_progress_bar(void)` ΓÇö clears progress state

## Control Flow Notes
Typical usage: `open_progress_dialog()` ΓåÆ repeatedly call `set_progress_dialog_message()` and/or `draw_progress_bar()` during operation ΓåÆ `close_progress_dialog()`. Used during initialization phases (network handshake, map loading, physics distribution, collection loading on PSP).

## External Dependencies
- No explicit includes visible in header
- Message IDs are string resource indices; actual strings defined in resource files (typical for Bungie/Marathon engine)
- Implementation (.cpp) likely includes platform-specific UI headers
