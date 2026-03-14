# Source_Files/shell_misc.cpp

## File Purpose
Manages cheat code detection and processing, XML configuration for cheats, and miscellaneous shell utilities including memory management and global idle tasks for the Aleph One game engine.

## Core Responsibilities
- Detects and processes cheat keyword sequences entered by the player
- Applies cheat effects (health, weapons, powerups, invincibility, etc.)
- Parses XML configuration for cheat enablement and custom keywords
- Provides memory management helpers for level transitions
- Executes global idle tasks (music, network, sound manager updates)
- Manages item distribution to the player via cheat activation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `keyword_data` | struct | Tag and keyword string pair for cheat matching |
| `XML_CheatsParser` | class (XML_ElementParser) | Parses `<cheats>` XML element to control `CheatsActive` flag and platform-specific modifiers |
| `XML_CheatKeywordParser` | class (XML_ElementParser) | Parses `<keyword>` XML elements to override cheat keyword strings |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `CheatsActive` | bool | global | Master flag enabling/disabling all cheat processing |
| `chat_input_mode` | bool | global | Indicates if player is currently typing chat input |
| `CheatCodeModMask` | unsigned short | global (mac-only) | Modifier key (e.g., controlKey) for typing cheat codes |
| `keyword_buffer` | char[MAXIMUM_KEYWORD_LENGTH+1] | static | Rolling buffer for last N characters typed; matched against cheat keywords |
| `keywords` | keyword_data[] | static | Lookup table mapping cheat tags to their keyword strings (e.g., "NRG" ΓåÆ health) |

## Key Functions / Methods

### handle_keyword
- **Signature:** `void handle_keyword(int tag)`
- **Purpose:** Executes the game action(s) corresponding to a matched cheat keyword.
- **Inputs:** `tag` ΓÇö one of the cheat tag enums (e.g., `_tag_health`, `_tag_invincible`).
- **Outputs/Return:** None (void).
- **Side effects:** Modifies player state (suit energy, oxygen, items, powerups); marks UI dirty; calls `save_game()` for the save cheat.
- **Calls:** `local_player` access, `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`, `process_player_powerup()`, `accelerate_monster()`, `AddItemsToPlayer()`, `AddOneItemToPlayer()`, `process_new_item_for_reloading()`, `update_interface()`, `save_game()`, `get_item_kind()`.
- **Notes:** Large switch statement with 20+ cheat cases. Some cheats grant weapons and ammo; others grant powerups. The `_tag_aslag` case grants all weapons/ammo and max health. No explicit "cheated" flag is set in final code (commented out).

### AddItemsToPlayer
- **Signature:** `void AddItemsToPlayer(short ItemType, short MaxNumber)`
- **Purpose:** Adds up to `MaxNumber` copies of an item to the player via repeated calls to `try_and_add_player_item()`.
- **Inputs:** `ItemType` ΓÇö item type constant, `MaxNumber` ΓÇö count to add.
- **Outputs/Return:** None.
- **Side effects:** Modifies player inventory via `try_and_add_player_item()`.
- **Calls:** `try_and_add_player_item()`.
- **Notes:** Simple loop; no checks for item limit or success status.

### AddOneItemToPlayer
- **Signature:** `void AddOneItemToPlayer(short ItemType, short MaxNumber)`
- **Purpose:** Ensures player has at least one copy of an item (if count < `MaxNumber`); adds one if needed.
- **Inputs:** `ItemType` ΓÇö item type constant, `MaxNumber` ΓÇö threshold.
- **Outputs/Return:** None.
- **Side effects:** May add item to inventory.
- **Calls:** `try_and_add_player_item()`.
- **Notes:** Used for weapons like magnum and shotgun where only 1ΓÇô2 copies are desired.

### process_keyword_key
- **Signature:** `int process_keyword_key(char key)`
- **Purpose:** Processes a single typed character; shifts the keyword buffer and checks for cheat matches at the buffer tail.
- **Inputs:** `key` ΓÇö character typed (converted to uppercase).
- **Outputs/Return:** Matched cheat tag (enum value) if a keyword is found; otherwise `NONE`.
- **Side effects:** Modifies `keyword_buffer` (circular shift). Clears buffer on match.
- **Calls:** `strcmp()`, `toupper()`, `memset()`.
- **Notes:** Sliding-window approach: buffer holds the last 20 characters, and each new character slides in from the right. Keyword matching is done at the **end** of the buffer.

### global_idle_proc
- **Signature:** `void global_idle_proc(void)`
- **Purpose:** Called regularly (during event loops) to perform housekeeping tasks.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `Music::instance()->Idle()`, `network_speaker_idle_proc()`, `network_microphone_idle_proc()`, `SoundManager::instance()->Idle()`.
- **Calls:** Idle methods on Music and SoundManager singletons; network idle functions.
- **Notes:** No-op if those systems are not initialized. Designed to be lightweight.

### free_and_unlock_memory
- **Signature:** `void free_and_unlock_memory(void)`
- **Purpose:** Frees up temporary memory when something important needs to happen (e.g., level load).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Stops all sounds; on Mac, purges and compacts memory.
- **Calls:** `SoundManager::instance()->StopAllSounds()`, `PurgeMem()`, `CompactMem()` (Mac only).
- **Notes:** Platform-specific; Mac uses Mac OS memory calls; SDL version only stops sounds.

### level_transition_malloc
- **Signature:** `void *level_transition_malloc(size_t size)`
- **Purpose:** Allocates memory during level transitions; falls back to unloading assets if initial malloc fails.
- **Inputs:** `size` ΓÇö bytes to allocate.
- **Outputs/Return:** Pointer to allocated block, or NULL if all fallbacks exhaust.
- **Side effects:** May unload sounds and collections if memory is tight.
- **Calls:** `malloc()`, `SoundManager::instance()->UnloadAllSounds()`, `unload_all_collections()`.
- **Notes:** Three-tier fallback: try malloc ΓåÆ unload sounds and retry ΓåÆ unload collections and retry. Helps avoid out-of-memory crashes during heavy asset loading.

### XML_CheatsParser::HandleAttribute
- **Signature:** `bool XML_CheatsParser::HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parses XML attributes for the `<cheats>` element.
- **Inputs:** `Tag` ΓÇö attribute name ("on" or "mac_keymod"), `Value` ΓÇö attribute value (string).
- **Outputs/Return:** `true` if attribute was recognized and parsed; `false` otherwise.
- **Side effects:** Sets `CheatsActive` if `Tag == "on"`; sets `CheatCodeModMask` if `Tag == "mac_keymod"` and platform is Mac.
- **Calls:** `StringsEqual()`, `ReadBooleanValueAsBool()`, `ReadUInt16Value()`, `UnrecognizedTag()`.
- **Notes:** Mac-specific handling for modifier key.

### XML_CheatKeywordParser::HandleString
- **Signature:** `bool XML_CheatKeywordParser::HandleString(const char *String, int Length)`
- **Purpose:** Sets the cheat keyword for a specific index (parsed from preceding `index` attribute).
- **Inputs:** `String` ΓÇö keyword text (UTF-8), `Length` ΓÇö byte length.
- **Outputs/Return:** `true`.
- **Side effects:** Updates `keywords[Index].keyword` with decoded and uppercased UTF-8 string.
- **Calls:** `DeUTF8_C()`, `toupper()`.
- **Notes:** Allows MML/XML to override default cheat keywords (e.g., change "NRG" to something else).

## Control Flow Notes
**Init/Load phase:**
- `Cheats_GetParser()` is called to return the root XML parser for cheat configuration.
- XML is parsed, setting `CheatsActive`, `CheatCodeModMask`, and custom keyword strings.

**Frame/Event loop:**
- `process_keyword_key()` is called for each character input (if not in menu/dialog).
- On match, `handle_keyword()` is called to apply the cheat effect.
- `global_idle_proc()` is called regularly to update Music, network, and sound systems.

**Level transitions:**
- `level_transition_malloc()` is called when loading map/asset data; uses memory fallbacks.

## External Dependencies
- **Notable includes:** `cseries.h` (core series utilities), `XML_ParseTreeRoot.h`, `interface.h`, `world.h`, `screen.h`, `map.h`, `shell.h`, `preferences.h`, `vbl.h`, `player.h`, `Music.h`, `items.h`, `network_sound.h`, `<ctype.h>`.
- **Defined elsewhere (called here):** `try_and_add_player_item()`, `process_new_item_for_reloading()`, `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`, `accelerate_monster()`, `network_speaker_idle_proc()`, `update_interface()`, `process_player_powerup()`, `save_game()`, `get_item_kind()`, `network_microphone_idle_proc()`, `local_player` (global), `local_player_index` (global), `dynamic_world` (global).
