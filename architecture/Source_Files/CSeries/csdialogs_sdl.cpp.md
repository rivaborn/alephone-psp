# Source_Files/CSeries/csdialogs_sdl.cpp

## File Purpose
Provides cross-platform dialog control compatibility functions between SDL and traditional Mac versions. Implements wrapper functions for querying and manipulating dialog widgets (toggles, selectors, text entries, number entries) with consistent interfaces.

## Core Responsibilities
- Hide/show dialog items by toggling widget enable/disable state
- Extract and insert numeric values from/into text entry widgets
- Query and modify selection control values (w_select widgets)
- Query and modify boolean control values (w_toggle widgets)
- Convert text between C-strings and Pascal-strings (pstrings) for text fields and static text
- Manage widget enable/disable state generically
- Provide null-safe wrapper functions (QQ_* family) that fail gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `widget` | class | Base class for all dialog widgets; provides enable/disable and generic interface |
| `w_number_entry` | class | Numeric input widget; subclass of w_text_entry |
| `w_text_entry` | class | Text input widget with buffer management |
| `w_static_text` | class | Static/read-only text display widget |
| `w_select` | class | Selection control (dropdown/list); manages index-based selection |
| `w_select_popup` | class | Popup variant of w_select; supports dynamic label updates |
| `w_toggle` | class | Boolean toggle (on/off) widget; subclass of w_select |
| `DialogPtr` / `DialogPTR` | typedef | Pointer to dialog object; owns widgets and manages layout |

## Global / File-Static State
None.

## Key Functions / Methods

### HideDialogItem / ShowDialogItem
- Signature: `void HideDialogItem(DialogPtr dialog, short item_index)` / `void ShowDialogItem(DialogPtr dialog, short item_index)`
- Purpose: Hide or show a dialog item by querying its widget and disabling/enabling it
- Inputs: `dialog` (dialog pointer), `item_index` (numeric widget ID)
- Outputs/Return: None (void)
- Side effects: Calls `widget::set_enabled(false)` or `widget::set_enabled(true)`; alters widget state
- Calls: `dialog::get_widget_by_id()`, `widget::set_enabled()`
- Notes: Fails silently if widget not found; semantic difference from Mac version (hidden widgets may still be interactive if enabled). File comment notes Mac version prevents interaction; SDL version does not.

### extract_number_from_text_item
- Signature: `long extract_number_from_text_item(DialogPtr dlg, short item)`
- Purpose: Extract integer value from a text entry widget
- Inputs: `dlg` (dialog pointer), `item` (widget ID for w_number_entry)
- Outputs/Return: `long` ΓÇö parsed integer from widget text buffer
- Side effects: None
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_number_entry*>()`, `w_number_entry::get_number()`
- Notes: Asserts that widget is non-null and correct type; caller must ensure item is a w_number_entry

### insert_number_into_text_item
- Signature: `void insert_number_into_text_item(DialogPtr dlg, short item, long number)`
- Purpose: Set integer value into a text entry widget
- Inputs: `dlg` (dialog pointer), `item` (widget ID for w_number_entry), `number` (value to set)
- Outputs/Return: None (void)
- Side effects: Updates widget text buffer via `w_number_entry::set_number()`
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_number_entry*>()`, `w_number_entry::set_number()`
- Notes: Asserts widget is non-null and correct type

### modify_selection_control
- Signature: `void modify_selection_control(DialogPtr inDialog, short inWhichItem, short inChangeEnable, short inChangeValue)`
- Purpose: Query and modify a selection widget's enable state and selected index
- Inputs: `inDialog` (dialog pointer), `inWhichItem` (widget ID), `inChangeEnable` (enable/disable flag or NONE), `inChangeValue` (1-based selection index or NONE)
- Outputs/Return: None (void)
- Side effects: Calls `widget::set_enabled()` and `w_select::set_selection()` if corresponding flags are not NONE
- Calls: `dialog::get_widget_by_id()`, `widget::set_enabled()`, `dynamic_cast<w_select*>()`, `w_select::set_selection()`
- Notes: Asserts widget is non-null; converts 1-based (Mac) to 0-based (internal) index via `inChangeValue - 1`; for w_select only (not w_toggle)

### modify_control_enabled
- Signature: `void modify_control_enabled(DialogPtr inDialog, short inWhichItem, short inChangeEnable)`
- Purpose: Enable or disable a dialog control
- Inputs: `inDialog` (dialog pointer), `inWhichItem` (widget ID), `inChangeEnable` (enable/disable flag or NONE)
- Outputs/Return: None (void)
- Side effects: Calls `widget::set_enabled()` if flag is not NONE
- Calls: `dialog::get_widget_by_id()`, `widget::set_enabled()`
- Notes: Generic control enable/disable; works on any widget type

### modify_boolean_control
- Signature: `void modify_boolean_control(DialogPtr inDialog, short inWhichItem, short inChangeEnable, short inChangeValue)`
- Purpose: Query and modify a toggle widget's enable state and selection
- Inputs: `inDialog` (dialog pointer), `inWhichItem` (widget ID), `inChangeEnable` (enable/disable flag or NONE), `inChangeValue` (0/1 selection or NONE)
- Outputs/Return: None (void)
- Side effects: Calls `widget::set_enabled()` and `w_toggle::set_selection()` if corresponding flags are not NONE
- Calls: `dialog::get_widget_by_id()`, `widget::set_enabled()`, `dynamic_cast<w_toggle*>()`, `w_toggle::set_selection()`
- Notes: Asserts widget is non-null; for w_toggle only (different indexing from w_select)

### get_selection_control_value
- Signature: `short get_selection_control_value(DialogPtr dialog, short which_control)`
- Purpose: Get the selected index from a selection widget
- Inputs: `dialog` (dialog pointer), `which_control` (widget ID for w_select)
- Outputs/Return: `short` ΓÇö 1-based selection index
- Side effects: None
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_select*>()`, `w_select::get_selection()`
- Notes: Asserts widget is non-null; converts 0-based internal to 1-based (Mac) index via `+ 1`

### get_boolean_control_value
- Signature: `bool get_boolean_control_value(DialogPtr dialog, short which_control)`
- Purpose: Get the boolean state from a toggle widget
- Inputs: `dialog` (dialog pointer), `which_control` (widget ID for w_toggle)
- Outputs/Return: `bool` ΓÇö toggle selection state
- Side effects: None
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_toggle*>()`, `w_toggle::get_selection()`
- Notes: Asserts widget is non-null

### copy_pstring_from_text_field
- Signature: `void copy_pstring_from_text_field(DialogPtr dialog, short item, unsigned char* pstring)`
- Purpose: Extract text from a text entry widget and convert to Pascal-string format
- Inputs: `dialog` (dialog pointer), `item` (widget ID for w_text_entry), `pstring` (output buffer)
- Outputs/Return: None (void); result written to `pstring` buffer
- Side effects: Modifies pstring buffer; calls `a1_c2pstr()` for in-place CΓåÆPascal conversion
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_text_entry*>()`, `w_text_entry::get_text()`, `strlen()`, `strncpy()`, `strcpy()`, `a1_c2pstr()`
- Notes: Asserts widget is non-null; truncates to 255 chars to fit pstring max length; caller responsible for buffer size

### copy_pstring_to_text_field
- Signature: `void copy_pstring_to_text_field(DialogPtr dialog, short item, const unsigned char* pstring)`
- Purpose: Convert Pascal-string to C-string and set it in a text entry widget
- Inputs: `dialog` (dialog pointer), `item` (widget ID for w_text_entry), `pstring` (Pascal-string input)
- Outputs/Return: None (void)
- Side effects: Updates widget text via `w_text_entry::set_text()`; allocates and frees temporary copy
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_text_entry*>()`, `pstrdup()`, `a1_p2cstr()`, `w_text_entry::set_text()`, `free()`
- Notes: Asserts widget is non-null; creates temporary duplicate for conversion

### copy_pstring_to_static_text
- Signature: `void copy_pstring_to_static_text(DialogPtr dialog, short item, const unsigned char* pstring)`
- Purpose: Convert Pascal-string to C-string and set it in a static text widget
- Inputs: `dialog` (dialog pointer), `item` (widget ID for w_static_text), `pstring` (Pascal-string input)
- Outputs/Return: None (void)
- Side effects: Updates static text display via `w_static_text::set_text()`
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<w_static_text*>()`, `pstrdup()`, `a1_p2cstr()`, `w_static_text::set_text()`, `free()`
- Notes: Asserts widget is non-null; pattern identical to copy_pstring_to_text_field

### QQ_* family (null-safe variants)
- Signature: Various (e.g., `bool QQ_control_exists()`, `void QQ_hide_control()`, `int QQ_get_selector_control_value()`)
- Purpose: Provide null-checking wrappers for common dialog operations; return sensible defaults on missing widgets
- Inputs: Typically `DialogPtr dlg` and `int item` (widget ID)
- Outputs/Return: Widget-type-dependent (bool, int, string, void)
- Side effects: Conditionally call widget methods if widget is found
- Calls: `dialog::get_widget_by_id()`, `dynamic_cast<>()`, and optional widget methods
- Notes: Fail gracefully (return false/0/empty string) if widget not found or wrong type; do not assert; useful for dialogs with optional/missing widgets

## Control Flow Notes
This file is a **utility/compatibility module** with no frame-based or event-driven control flow. Functions are called synchronously during:
1. **Dialog initialization**: Set initial control values/states
2. **Dialog manipulation**: Query or update controls in response to user actions or programmatic changes
3. **Dialog teardown**: Not explicitly handled here

Functions run in the context of the dialog system (defined in sdl_dialogs.h); they do not own event dispatch or rendering.

## External Dependencies
- `cseries.h` ΓÇö Base engine definitions (types, macros)
- `sdl_dialogs.h` ΓÇö Dialog class definition; provides `DialogPtr`, `dialog::get_widget_by_id()`
- `sdl_widgets.h` ΓÇö Widget class hierarchy (w_number_entry, w_select, w_toggle, w_text_entry, w_static_text, w_select_popup)
- C standard library ΓÇö `<string.h>` functions (strlen, strncpy, strcpy)
- Game-local string functions ΓÇö `pstrdup()`, `a1_c2pstr()`, `a1_p2cstr()` (PascalΓåöC conversion); `build_stringvector_from_stringset()` (defined elsewhere)
