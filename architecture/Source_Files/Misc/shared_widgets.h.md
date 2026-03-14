# Source_Files/Misc/shared_widgets.h

## File Purpose

Cross-platform header providing preference binding adapters and chat history management for dialog systems. Abstracts platform-specific widget implementations (SDL vs. Carbon) and offers high-level preference classes that integrate with a data binding system.

## Core Responsibilities

- Define preference binding adapters (PStringPref, CStringPref, BoolPref, BitPref, Int16Pref, FilePref) to bind game configuration values to UI widgets
- Provide platform-agnostic chat history storage and notification system
- Bridge chat history state to UI components via the observer pattern (ChatHistory ΓåÆ ColorfulChatWidget)
- Conditionally include platform-specific widget headers based on SDL vs. Carbon build target

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| PStringPref | class | Binds Pascal strings (Pstring) to std::string for preference serialization |
| CStringPref | class | Binds C strings to std::string for preference serialization |
| BoolPref | class | Binds bool reference for togglable preferences |
| BitPref | class | Binds individual bits in a uint16 mask to bool (with optional inversion) |
| Int16Pref | class | Binds int16 value to int for preference storage |
| FilePref | class | Binds file paths (FSSpec on Mac, char* on other platforms) to FileSpecifier |
| ChatHistory | class | Stores vector of ColoredChatEntry; notifies observer on append/clear |
| ColorfulChatWidget | class | Observer that syncs ChatHistory changes to w_colorful_chat UI widget |

## Global / File-Static State

None.

## Key Functions / Methods

### PStringPref::bind_export
- Signature: `virtual std::string bind_export()`
- Purpose: Export Pstring preference to std::string
- Inputs: None (uses internal m_pref)
- Outputs/Return: std::string
- Calls: `pstring_to_string(m_pref)`
- Notes: Assumes m_pref is valid Pstring (unsigned char*)

### PStringPref::bind_import
- Signature: `virtual void bind_import(std::string s)`
- Purpose: Import std::string into Pstring preference
- Inputs: s (std::string)
- Outputs/Return: None
- Side effects: Modifies m_pref; truncates/pads to m_length
- Calls: `copy_string_to_pstring()`

### BitPref::bind_export
- Signature: `virtual bool bind_export()`
- Purpose: Extract bit value from masked uint16 with optional negation
- Inputs: None (uses internal m_pref, m_mask, m_invert)
- Outputs/Return: bool (bit extracted and optionally inverted)
- Notes: Handles bit-level masking and boolean inversion in single method

### BitPref::bind_import
- Signature: `virtual void bind_import(bool value)`
- Purpose: Set bit value in masked uint16 with optional negation
- Inputs: value (bool)
- Outputs/Return: None
- Side effects: Modifies m_pref using bitwise OR/AND with m_mask
- Notes: Inverts value before masking if m_invert is true

### ChatHistory::append
- Signature: `void append(const ColoredChatEntry& e)`
- Purpose: Add chat entry to history and notify observer
- Inputs: e (ColoredChatEntry)
- Outputs/Return: None
- Side effects: Pushes to m_history vector; calls m_notificationAdapterΓåÆcontentAdded if set

### ChatHistory::setObserver
- Signature: `void setObserver(NotificationAdapter* notificationAdapter)`
- Purpose: Register observer for history changes
- Inputs: notificationAdapter (pointer to observer)
- Outputs/Return: None
- Notes: Observer lifecycle managed by caller; no ownership taken

### ColorfulChatWidget::attachHistory
- Signature: `void attachHistory(ChatHistory* history)`
- Purpose: Link this widget to a ChatHistory source
- Inputs: history (ChatHistory pointer)
- Outputs/Return: None
- Side effects: Stores history pointer; registers self as observer
- Calls: `historyΓåÆsetObserver(this)`

### ColorfulChatWidget::contentAdded
- Signature: `virtual void contentAdded(const ColoredChatEntry& e)`
- Purpose: Handle new chat entry from observed ChatHistory
- Inputs: e (ColoredChatEntry)
- Outputs/Return: None
- Side effects: Calls `m_componentWidgetΓåÆAppend(e)`

## Control Flow Notes

**Initialization:**
- Preference classes (PStringPref, etc.) are instantiated during dialog setup, typically wrapping existing game settings storage
- ChatHistory is created once per game session; ColorfulChatWidget is created when chat UI is needed
- ColorfulChatWidget calls attachHistory() to subscribe to ChatHistory notifications

**Runtime:**
- Chat entries are appended to ChatHistory, which notifies ColorfulChatWidget
- ColorfulChatWidget forwards updates to w_colorful_chat (SDL) or equivalent Carbon widget
- Preference bindings act as two-way bridges: UI reads/writes via bind_export/bind_import

**Platform-specific behavior:**
- Header guards select sdl_widgets.h or carbon_widgets.h at compile time
- FilePref has two implementations: FSSpec-based (Mac) vs. path string (other platforms)

## External Dependencies

- **Notable includes:**
  - `cseries.h` ΓÇô Core types (std::string, uint16, int16)
  - `sdl_widgets.h` or `carbon_widgets.h` (conditional) ΓÇô Platform-specific widget classes
  - `binders.h` ΓÇô Base Bindable<T> template and Binder infrastructure

- **Defined elsewhere:**
  - `pstring_to_string()`, `copy_string_to_pstring()`, `copy_string_to_cstring()` ΓÇô String conversion utilities
  - `ColoredChatEntry` struct ΓÇô Defined in this file but used by ChatHistory
  - `w_colorful_chat` ΓÇô Widget class (defined in sdl_widgets.h or carbon_widgets.h)
  - `FileSpecifier` ΓÇô File path abstraction (likely defined in FileHandler.h)
