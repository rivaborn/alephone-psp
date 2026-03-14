# Source_Files/Misc/preferences.cpp

## File Purpose
Manages game preferences (settings) for the Aleph One Marathon engine. Provides XML-based persistence, parsing, and UI dialogs for player, graphics, sound, network, input, and environment preferences.

## Core Responsibilities
- Load and parse preferences from XML preference files
- Display preference configuration dialogs (player, graphics, sound, controls, environment, crosshairs)
- Save modified preferences back to XML format
- Assemble XML parser hierarchy for preferences deserialization
- Adapt platform-specific operations (user name retrieval, network detection)
- Manage preference data structures across graphics, sound, network, player, and input subsystems

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CrosshairPref | class | Data binding adapter for crosshair thickness (int16 Γåö int) |
| ColorComponentPref | class | Data binding adapter for 16-bit color components (uint16 Γåö int) |
| OpacityPref | class | Data binding adapter for float opacity (float Γåö int) |
| XML_GraphicsPrefsParser | class | Parses graphics preference XML attributes |
| XML_ChaseCamPrefsParser | class | Parses chase camera preference XML attributes |
| XML_CrosshairsPrefsParser | class | Parses crosshair preference XML attributes |
| XML_PlayerPrefsParser | class | Parses player preference XML attributes |
| XML_MouseButtonPrefsParser | class | Parses mouse button action mappings |
| XML_KeyPrefsParser | class | Parses key binding definitions |
| XML_InputPrefsParser | class | Parses input device and sensitivity settings |
| XML_SoundPrefsParser | class | Parses sound volume and channel settings |
| XML_NetworkPrefsParser | class | Parses network game and metaserver settings |
| XML_EnvironmentPrefsParser | class | Parses map/physics/shapes file paths and checksums |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| PrefsRootParser | XML_ElementParser | static | Root node for preference XML parsing |
| MarathonPrefsParser | XML_ElementParser | static | Canonical root element ("mara_prefs") in XML |
| XML_DataBlockLoader | XML_DataBlock | static | Interprets read-in preference files |
| PrefsInited | bool | static | Tracks whether preferences have been initialized |
| sPasswordMask | const char[] | static | Obfuscation key for metaserver password encoding ("reverof nohtaram") |
| sNetworkGameProtocolNames | const char*[] | static | Maps protocol indices to names ("ring", "star") |
| graphics_preferences | struct* | global | Active graphics preference data |
| serial_preferences | struct* | global | Serial number / licensing data |
| network_preferences | struct* | global | Network game configuration |
| player_preferences | struct* | global | Player profile (name, color, difficulty) |
| input_preferences | struct* | global | Input device and key binding configuration |
| sound_preferences | SoundManager::Parameters* | global | Audio settings (volume, channels, rate) |
| environment_preferences | struct* | global | Map/resource file paths and checksums |

## Key Functions / Methods

### handle_preferences()
- **Signature:** `void handle_preferences(void)`
- **Purpose:** Main entry point; displays the primary preferences dialog with buttons for all preference categories.
- **Inputs:** None (retrieves global preference pointers).
- **Outputs/Return:** None (modifies global preference state and UI).
- **Side effects:** Calls `write_preferences()` before showing dialogs; loads 8-bit color table if needed; displays main menu after dialog closes.
- **Calls:** `write_preferences()`, `build_8bit_system_color_table()`, SDL video surface queries, dialog constructors.
- **Notes:** Creates a vertical layout with title and buttons for Player, Graphics, Sound, Controls, Environment settings. Clears screen before running dialog.

### player_dialog()
- **Signature:** `static void player_dialog(void *arg)`
- **Purpose:** Displays player configuration dialog (name, color, team, difficulty, metaserver credentials, chat colors).
- **Inputs:** `arg` (parent dialog pointer, cast from void*).
- **Outputs/Return:** None (modifies player_preferences and network_preferences globals).
- **Side effects:** Writes preferences if any changes detected; calls crosshair_dialog when "CROSSHAIR" button pressed.
- **Calls:** `copy_pstring_to_text_field()`, `copy_pstring_from_text_field()`, dialog widget constructors, `write_preferences()`.
- **Notes:** Compares old vs. new values to detect changes; converts password strings safely; disables login/password fields when "Guest" toggle is enabled.

### crosshair_dialog()
- **Signature:** `static void crosshair_dialog(void *arg)`
- **Purpose:** Advanced crosshair customization (shape, thickness, gap, size, color, opacity).
- **Inputs:** `arg` (parent dialog pointer).
- **Outputs/Return:** None (modifies player_preferences->Crosshairs).
- **Side effects:** Saves old crosshair state for undo on cancel; clears PreCalced flag and writes preferences on accept.
- **Calls:** Widget constructors, `crosshair_binders->migrate_*()` for data binding synchronization.
- **Notes:** Uses auto_ptr<BinderSet> to manage bidirectional data binding. Clears screen before running dialog.

### get_name_from_system()
- **Signature:** `static void get_name_from_system(unsigned char *outName)`
- **Purpose:** Retrieves the current user's login name from the OS.
- **Inputs:** `outName` (destination buffer for Pstring name).
- **Outputs/Return:** None (fills outName with OS user name or "Bob User" default).
- **Side effects:** Buffer overflow risk (caller must provide adequate space); calls `a1_c2pstr()` for Pascal string conversion.
- **Calls:** Platform-specific: `getlogin()` (Unix), `GetUserName()` (Windows).
- **Notes:** Strips illegal filename characters on Windows; falls back to "Bob User" if no name found.

### XML_*PrefsParser::HandleAttribute() methods
- **Signature:** `bool XML_*PrefsParser::HandleAttribute(const char *Tag, const char *Value)` (various subclasses)
- **Purpose:** Parse individual XML preference attributes and update global preference structures.
- **Inputs:** `Tag` (attribute name), `Value` (string value from XML).
- **Outputs/Return:** `bool` (success/failure).
- **Side effects:** Modifies global preference pointers (graphics_preferences, player_preferences, etc.).
- **Calls:** Helper functions like `ReadInt16Value()`, `ReadBooleanValue()`, `ReadFloatValue()`, `DeUTF8_C()`, `StringsEqual()`.
- **Notes:** Each parser is tag-specific (graphics, player, crosshairs, etc.). Password is XOR-decoded using sPasswordMask.

### SetupPrefsParseTree()
- **Signature:** `void SetupPrefsParseTree()`
- **Purpose:** Assembles the XML parser hierarchy for preferences deserialization.
- **Inputs:** None (configures static parser instances).
- **Outputs/Return:** None (initializes parent-child parser relationships).
- **Side effects:** None (pure setup, no I/O).
- **Calls:** `AddChild()` on parser instances.
- **Notes:** Constructs tree with PrefsRootParser ΓåÆ MarathonPrefsParser ΓåÆ category-specific parsers (Graphics, Player, Input, Sound, Network, Environment). Disables networking parsers if DISABLE_NETWORKING defined.

### ethernet_active()
- **Signature:** `static bool ethernet_active(void)`
- **Purpose:** Checks network availability (stub implementation).
- **Inputs:** None.
- **Outputs/Return:** Always returns `true`.
- **Side effects:** None.
- **Notes:** This is a placeholder; actual network detection may be needed.

## Control Flow Notes
Game preferences are typically loaded during initialization via `read_preferences()` and `SetupPrefsParseTree()` (XML parsing setup). When the user accesses the preferences menu, `handle_preferences()` is called, which displays a dialog. User selections trigger category-specific dialogs (player_dialog, graphics_dialog, etc.). On acceptance, `write_preferences()` persists changes. The XML parser hierarchy is used to deserialize the preference file into the global preference data structures.

## External Dependencies
- **Notable includes:** cseries.h (basic types/macros), FileHandler.h (file I/O), map.h (world structures), shell.h (screen_mode), interface.h (game state), SoundManager.h (audio parameters), ISp_Support.h (input device handling), XML_ElementParser.h / XML_DataBlock.h / ColorParser.h (XML parsing), network headers (protocol handling).
- **Defined elsewhere:** `write_preferences()`, `read_preferences()`, dialog functions (graphics_dialog, sound_dialog, etc.), color table builders, SDL surface management, player/input/sound preference initializers and validators.
