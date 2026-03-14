# Source_Files/Network/Metaserver/SdlMetaserverClientUi.cpp

## File Purpose

SDL-specialized UI implementation for the metaserver client that allows players to locate and join network games. Builds a modal dialog with player/game lists, chat interface, and game information display, then manages its lifetime and event pumping.

## Core Responsibilities

- **Dialog Construction**: Assembles a complex multi-widget SDL dialog layout (title, player/game lists, chat, buttons)
- **Widget Wrapping**: Bridges raw SDL widgets to higher-level abstraction layers (PlayerListWidget, GameListWidget, etc.)
- **Modal Dialog Lifecycle**: Manages initialization, running, and cleanup of the metaserver UI dialog
- **Idle Processing**: Implements `pump()` callback to refresh game list and monitor connection status during dialog idle time
- **Game Information Display**: Creates and manages nested modal dialog for displaying detailed game metadata
- **User Input Handling**: Connects button clicks, game selections, and chat input to game logic via callbacks

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SdlMetaserverClientUi` | class | Main UI implementation; inherits from `MetaserverClientUi` abstract interface |
| `dialog` | class | SDL modal dialog container; holds widgets and manages event/render loops |
| `ColorfulChatEntry` | struct | Chat message with type, sender, color, and text (from sdl_widgets.h) |
| `GameListMessage::GameListEntry` | struct | Game server details: host player, map, difficulty, options, etc. (from network headers) |
| `MetaserverPlayerInfo` | struct | Player metadata: name, ID, color, status (from network headers) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverClient` | `MetaserverClient*` | global (external) | Global metaserver client; queries game list, sends chat, joins games |
| `m_disconnected` | `bool` | member (SdlMetaserverClientUi) | Flag: connection to room lost; used to detect and alert user |

## Key Functions / Methods

### Constructor `SdlMetaserverClientUi()`
- **Signature**: `SdlMetaserverClientUi()`
- **Purpose**: Initialize and construct the complete metaserver dialog UI
- **Inputs**: None
- **Outputs/Return**: None (object constructed)
- **Side effects**: 
  - Allocates and chains ~20 SDL widget objects (placers, buttons, lists, text entries)
  - Populates `d` (dialog member) with widget placer tree
  - Stores raw widget pointers and wraps them in abstraction layer objects (m_gamesInRoomWidget, m_chatWidget, etc.)
  - Activates chat entry widget for focus
- **Calls**: `vertical_placer::dual_add()`, `table_placer::dual_add()`, `horizontal_placer::dual_add()`, `w_tiny_button::set_enabled()`, various widget constructors
- **Notes**: 
  - Uses Boost `bind()` to connect `GameSelected()` callback to games_in_room_w
  - Layout is fixed and built at compile-time (no dynamic resizing of structure)
  - Spacing and theme constants sourced from `get_theme_space()`

### `Run()`
- **Signature**: `int Run()`
- **Purpose**: Display the dialog modally and block until user dismisses it
- **Inputs**: None
- **Outputs/Return**: Dialog result code (ΓêÆ1 if user clicked cancel, 0+ if OK)
- **Side effects**: 
  - Sets `pump()` as the dialog's idle callback via `set_processing_function()`
  - Blocks in `d.run()` until dialog finishes
  - On cancel (result ΓêÆ1), clears `m_joinAddress` struct
- **Calls**: `dialog::set_processing_function()`, `dialog::run()`
- **Notes**: Dialog stays responsive to events while pump fires on idle

### `Stop()`
- **Signature**: `void Stop()`
- **Purpose**: Terminate the modal dialog loop (triggered by connection loss or user cancel)
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Calls `dialog_ok()` to signal dialog to close and return
- **Calls**: `dialog_ok()`
- **Notes**: SynchronousΓÇöreturns immediately; actual dialog close happens on next event loop

### `InfoClicked()`
- **Signature**: `void InfoClicked()`
- **Purpose**: Display a nested modal dialog with detailed game information (map, difficulty, players, options, etc.)
- **Inputs**: None (reads selected game via `gMetaserverClient->game_target()`)
- **Outputs/Return**: None
- **Side effects**: 
  - Creates ephemeral `info_dialog` and widget tree; destroys on scope exit
  - If user clicks JOIN in info dialog, calls `JoinGame(*game)` (defined elsewhere)
  - If user cancels or game no longer exists, calls `GameSelected(*game)` to deselect
- **Calls**: 
  - `gMetaserverClient->find_game()`, `find_player()`
  - `TS_GetCString()` (localization)
  - `Scenario::instance()->IsCompatible()` (scenario check)
  - `dialog::run()`, `dialog::set_processing_function()`
- **Notes**: 
  - Large nested if-tree to construct table rows for game metadata (15+ parameters)
  - Creates toggle widgets for game flags (Aliens, Teams, Disable Motion Sensor, etc.) but disables them (read-only)
  - Handles conversion of game type enum (subtracts 1 if > 5) for string lookup
  - Constructs time limit string manually (minutes from ticks); uses "No" for unlimited/invalid

### `pump()`
- **Signature**: `void pump(dialog* d)`
- **Purpose**: Periodic idle callback to refresh game list and check connection status
- **Inputs**: `dialog* d` ΓÇô pointer to calling dialog
- **Outputs/Return**: None
- **Side effects**: 
  - Every 5000ms: calls `games_in_room_w->refresh()` to mark widget dirty and redraw
  - Every call: polls `gMetaserverClient->isConnected()` and alerts user if disconnected
  - On disconnect: sets `m_disconnected` flag and calls `Stop()` to close dialog
- **Calls**: `SDL_GetTicks()`, `w_games_in_room::refresh()`, `MetaserverClient::pumpAll()`, `alert_user()`, `Stop()`
- **Notes**: 
  - Static `last_update` tracks time since last refresh (initialization is undefined; first call likely refreshes immediately)
  - `MetaserverClient::pumpAll()` is socket/network pumpΓÇöprocesses inbound messages

### Factory Function `MetaserverClientUi::Create()`
- **Signature**: `auto_ptr<MetaserverClientUi> MetaserverClientUi::Create()`
- **Purpose**: Factory function to construct and return a new SDL-based metaserver UI (virtual implementation)
- **Inputs**: None
- **Outputs/Return**: Dynamically allocated `SdlMetaserverClientUi` wrapped in `auto_ptr`
- **Side effects**: Heap allocation
- **Calls**: `new SdlMetaserverClientUi()`
- **Notes**: Implements abstract virtual in `MetaserverClientUi` base class; allows decoupling of UI choice from core

## Control Flow Notes

- **Initialization**: Constructor called once per session; builds entire UI tree synchronously
- **Runtime**: `Run()` enters modal event loop; dialog calls `pump()` on idle (~10ms intervals typical in SDL)
- **Update Phase**: `pump()` checks connection status and refreshes game list every 5 seconds
- **User Interaction**: Button callbacks and list selection callbacks (bound via Boost) invoke game logic (join, chat, etc.)
- **Shutdown**: Dialog exits when user clicks CANCEL, JOIN, or connection is lost; wrapper objects cleaned up on dialog destruction

## External Dependencies

**Notable Includes/Imports**:
- `sdl_dialogs.h`, `sdl_fonts.h`, `sdl_widgets.h` ΓÇô SDL UI framework (dialogs, widgets, layout)
- `network_metaserver.h` ΓÇô metaserver client API; queries games/players
- `network_dialog_widgets_sdl.h` ΓÇô specialized metaserver list widgets (w_games_in_room, w_players_in_room)
- `screen_drawing.h` ΓÇô font/color drawing utilities
- `interface.h` ΓÇô game window/UI state (set_drawing_clip_rectangle)
- `TextStrings.h` ΓÇô localization (TS_GetCString for difficulty/game type strings)
- `boost/function.hpp`, `boost/bind.hpp` ΓÇô function binding for callbacks
- `<sstream>`, `<algorithm>` ΓÇô C++ utilities

**External Symbols (defined elsewhere)**:
- `gMetaserverClient` ΓÇô global metaserver client instance (network_metaserver.cpp)
- `GameSelected()`, `JoinGame()` ΓÇô callbacks for game list interaction (metaserver_dialogs.cpp or similar)
- `delete_widgets()` ΓÇô cleanup helper (likely from MetaserverClientUi base or dialog system)
- `Scenario::instance()` ΓÇô singleton map/scenario manager (Scenario.cpp or similar)
- `alert_user()`, `dialog_ok()`, `dialog_cancel()` ΓÇô dialog utilities (sdl_dialogs.cpp)
- `get_theme_*()`, `get_interface_*()` ΓÇô theme/UI constants (sdl_dialogs.cpp, screen_drawing.cpp)
