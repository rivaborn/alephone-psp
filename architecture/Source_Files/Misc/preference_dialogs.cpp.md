# Source_Files/Misc/preference_dialogs.cpp

## File Purpose
Implements OpenGL graphics preference dialogs for Aleph One game engine. Provides UI for configuring graphics quality, texture settings, filtering, FSAA, and anisotropy. Uses a binding system to synchronize UI widgets with persistent graphics preferences.

## Core Responsibilities
- Build and manage OpenGL preference dialogs with tabbed General/Advanced sections
- Adapt internal preference values to/from UI widget representations via Bindable converters
- Bidirectional synchronization between UI state and `graphics_preferences` via BinderSet
- Create platform-specific dialog implementations (SDL) using placer-based layout system
- Handle texture quality/resolution/depth presets and filter options
- Manage lifecycle of dialog widgets and preference binding

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TexQualityPref` | class | Converts texture max size (128/256/512...) to/from UI log scale (0-4) |
| `ColourPref` | class | Passes through RGBColor values without conversion |
| `FilterPref` | class | Maps filter indices (1,3,5...) to/from UI popup indices (0,1,2...) |
| `TimesTwoPref` | class | Divides/multiplies int16 by 2 for FSAA sample counts |
| `AnisotropyPref` | class | Converts float anisotropy level to/from log scale |
| `OpenGLDialog` | class | Abstract base; defines widget members and `OpenGLPrefsByRunning()` orchestrator |
| `SdlOpenGLDialog` | class | SDL concrete implementation; builds UI hierarchy in constructor |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | extern global | Active graphics config; preferences are read/written via this pointer |
| `filter_labels` | `const char*[4]` | static | Label strings for texture filter dropdown ("Linear", "Bilinear", "Trilinear", NULL) |

## Key Functions / Methods

### TexQualityPref::bind_export / bind_import
- **Purpose:** Convert internal texture MaxSize (power-of-2 values) to/from UI popup index (0ΓÇô4 scale)
- **Inputs:** bind_export returns m_pref; bind_import takes UI index
- **Outputs/Return:** bind_export returns result (log scale); bind_import sets m_pref
- **Side effects:** None (data only)
- **Notes:** Right-shifts to find log, so 256 ΓåÆ index 2; inverse computes `m_normal << (value - 1)`

### AnisotropyPref::bind_export / bind_import
- **Purpose:** Convert float anisotropy level to/from UI slider index (0ΓÇô6 for 1ΓÇô64x)
- **Inputs:** bind_export reads m_pref (float); bind_import takes index
- **Outputs/Return:** Index is log(anisotropy); inverse is `1 << (value - 1)` or 0.0
- **Side effects:** None (data only)
- **Notes:** Handles edge case value==0 ΓåÆ 0.0

### FilterPref, TimesTwoPref::bind_export / bind_import
- **Purpose:** Linear scale mappings: FilterPref scales by 2 with offset +1; TimesTwoPref scales by 2
- **Notes:** FilterPref used for GL filter types (GL_LINEAR, GL_LINEAR_MIPMAP_LINEAR, etc.)

### OpenGLDialog::OpenGLPrefsByRunning
- **Signature:** `void OpenGLPrefsByRunning()`
- **Purpose:** Main dialog orchestrator; binds all widgets to preferences, runs dialog, migrates/saves on accept
- **Inputs:** None (reads `graphics_preferences` directly)
- **Outputs/Return:** None; modifies preferences and calls `write_preferences()` if accepted
- **Side effects:** 
  - Creates BinderSet with widget-to-preference adapters
  - Calls `migrate_all_second_to_first()` to load prefs into UI
  - Calls `Run()` (blocking dialog loop)
  - Calls `migrate_all_first_to_second()` if accepted, then `write_preferences()`
- **Calls:** `binders.insert<T>()`, `binders.migrate_all_second_to_first()`, `Run()`, `migrate_all_first_to_second()`, `write_preferences()`
- **Notes:** Template instantiation for multiple Bindable types (bool, int, RGBColor); handles Z-buffer, fog, effects, FSAA, texture quality/resolution/depth, filters, anisotropy, void color

### SdlOpenGLDialog::SdlOpenGLDialog (constructor)
- **Signature:** `SdlOpenGLDialog()`
- **Purpose:** Build complete UI hierarchy using placer-based layout; wrap raw widgets in adapter classes
- **Inputs:** None
- **Outputs/Return:** Fully constructed dialog with m_dialog, m_tabs, and all m_*Widget members
- **Side effects:** 
  - Allocates vertical/table/horizontal placers
  - Creates w_toggle, w_select_popup, w_slider, w_select, w_button, w_label, w_static_text, w_spacer widgets
  - Wraps in ButtonWidget, ToggleWidget, PopupSelectorWidget, SliderSelectorWidget, SelectSelectorWidget
  - Sets callbacks for OK/Cancel buttons (bound to `Stop()`)
  - Configures tab placer with "GENERAL" and "ADVANCED" tabs
- **Calls:** Extensive widget factory and layout calls; `set_labels()`, `set_callback()`, `associate_label()`, etc.
- **Notes:** 
  - m_colourTheVoidWidget and m_voidColourWidget set to nullptr (not used in SDL version)
  - m_modelQualityWidget created separately for 3D model skin quality
  - All texture types (walls, landscapes, sprites, weapons) get quality/resolution/depth controls

### SdlOpenGLDialog::Run / Stop
- **Signature:** `virtual bool Run()` / `virtual void Stop(bool result)`
- **Purpose:** Run blocking dialog loop; quit dialog with result code
- **Inputs/Outputs:** Run returns true if accepted (result==0); Stop sets quit code (0 for accept, -1 for cancel)
- **Side effects:** Blocking UI event loop; writes m_dialog state
- **Calls:** `m_dialog.run()`, `m_dialog.quit()`

### SdlOpenGLDialog::choose_generic_tab / choose_advanced_tab
- **Purpose:** Switch active tab in response to button click (commented out in current code)
- **Notes:** Trivial; just call `m_tabs->choose_tab()` and redraw

### OpenGLDialog::Create (factory)
- **Signature:** `static auto_ptr<OpenGLDialog> Create()`
- **Purpose:** Factory method; returns SdlOpenGLDialog instance wrapped in auto_ptr
- **Notes:** Abstract factory pattern; concrete type is determined at link time

## Control Flow Notes
**Initialization ΓåÆ Preferences Editing ΓåÆ Persistence:**
1. User calls `OpenGLPrefsByRunning()` (from preferences UI)
2. BinderSet binds each widget to a Bindable adapter ΓåÆ preference field
3. `migrate_all_second_to_first()` loads current pref values into widgets
4. `Run()` blocks until user accepts/cancels
5. If accepted, `migrate_all_first_to_second()` copies widget values back to preferences
6. `write_preferences()` persists changes to disk

Dialog runs at the OS event-loop level (blocking call); no frame-based updates involved.

## External Dependencies
- **Key includes:** `preference_dialogs.h`, `preferences.h` (graphics_preferences struct), `binders.h` (Bindable, BinderSet), `OGL_Setup.h` (OGL_ConfigureData, texture types, flags)
- **External symbols used:**
  - `graphics_preferences` (global extern)
  - `write_preferences()` (defined elsewhere)
  - Widget classes: `w_toggle`, `w_select_popup`, `w_slider`, `w_select`, `w_button`, `w_label`, `w_static_text`, `w_spacer`, `tab_placer`, `dialog`, etc. (from widget framework, likely `shared_widgets.h`)
  - Placer classes: `vertical_placer`, `horizontal_placer`, `table_placer` (layout system)
  - Adapter widget classes: `ButtonWidget`, `ToggleWidget`, `PopupSelectorWidget`, `SliderSelectorWidget`, `SelectSelectorWidget` (from `shared_widgets.h`)
  - `boost::bind` (for callback binding)
