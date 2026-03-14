# Source_Files/CSeries/csdialogs.h

## File Purpose
Cross-platform dialog box abstraction header providing a unified API for manipulating UI controls on both Mac OS Toolbox and SDL-based implementations. Bridges platform-specific differences in dialog handling and control numbering conventions between Mac and non-Mac platforms.

## Core Responsibilities
- Define platform-agnostic dialog pointer types (DialogPtr, DialogPTR) with conditional compilation
- Declare common cross-platform control manipulation functions (QQ_* functions)
- Provide specialized control-type wrappers (boolean, selector/popup, text fields)
- Define control activity state constants (CONTROL_INACTIVE, CONTROL_ACTIVE)
- Offer Mac OS-specific dialog utilities (event filtering, window frame manipulation, control modification)
- Handle differences in control indexing between Mac OS (1-based) and SDL implementations (0-based)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| DialogPtr | typedef | Platform-specific dialog handle (DialogPtr on Mac, dialog* on SDL) |
| DialogPTR | typedef | Variant of DialogPtr used in some contexts |
| dialog | forward class | SDL-side opaque dialog class (defined elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### QQ_control_exists, QQ_hide_control, QQ_show_control, QQ_set_control_activity
- Purpose: Check control existence and manage visibility/activity state
- Inputs: dialog pointer, control item ID, (optional) boolean state
- Outputs/Return: bool (exists), void (others)
- Notes: Part of unified cross-platform control API (QQ_* prefix)

### QQ_get_boolean_control_value / QQ_set_boolean_control_value
- Purpose: Get/set checkbox or toggle control state
- Inputs: dialog pointer, item ID, (optional) boolean value
- Outputs/Return: bool (get) / void (set)
- Notes: Abstracts Mac vs SDL indexing differences

### QQ_get_selector_control_value / QQ_set_selector_control_value
- Purpose: Get/set popup/dropdown selection
- Inputs: dialog pointer, item ID, (optional) value; labels optionally from vector or stringset ID
- Outputs/Return: int (get) / void (set)
- Side effects (set_labels): Updates control label list

### QQ_copy_string_from_text_control / QQ_extract_number_from_text_control
- Purpose: Extract text or numeric value from text field
- Inputs: dialog pointer, item ID
- Outputs/Return: std::string or long

### QQ_copy_string_to_text_control / QQ_insert_number_into_text_control
- Purpose: Set text or numeric value in text field
- Inputs: dialog pointer, item ID, value (string/number)
- Outputs/Return: void

### Mac-specific: modify_control, modify_selection_control, modify_boolean_control
- Purpose: Alter control state (hilite/value) on Mac OS
- Inputs: dialog pointer, item ID, hilite state, (optional) value
- Outputs/Return: void
- Notes: Mac OS Toolbox API; on non-Mac, specialized versions are separately declared (not inlined)

### Mac-specific: myGetNewDialog, get_general_filter_upp
- Purpose: Create new modal dialog and obtain event filter UPP (Universal Procedure Pointer)
- Notes: Mac-only utilities for dialog initialization and event handling

## Control Flow Notes
This header is purely declarativeΓÇöit defines interfaces used by dialog/UI management code. Control flow occurs in implementation files (csdialogs.cpp or platform-specific variants) and in code that calls these functions during game menu/UI presentation.

## External Dependencies
- **Includes**: `<string>` (C++ standard), `<vector>` (C++ standard)
- **Conditional Mac includes**: `<Dialogs.h>` (Mac OS Toolbox, included via SDL_RFORK_HACK macro workaround)
- **Platform detection**: Preprocessor flags `mac`, `SDL_RFORK_HACK`, `USES_NIBS`, `TARGET_API_MAC_CARBON` determine which declarations/types are active
- **Defined elsewhere**: `dialog` class (SDL implementation), `ListHandle`, `ControlHandle`, `WindowRef`, `WindowPtr`, `EventRecord`, `Rect`, `DialogPtr` (native Mac types), various UPP types (Mac toolbox)
