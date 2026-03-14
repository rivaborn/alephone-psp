# Source_Files/Network/Metaserver/metaserver_dialogs.h

## File Purpose
Defines UI classes for the metaserver client dialog system. Provides abstractions for displaying available games, players, and chat messages from a metaserver, with platform-agnostic interfaces and factory-based concrete implementations.

## Core Responsibilities
- Define abstract base class (`MetaserverClientUi`) for metaserver UI with factory instantiation
- Implement notification adapter to bridge metaserver events to UI updates
- Manage game announcement lifecycle via `GameAvailableMetaserverAnnouncer`
- Handle player/game selection, chat input, muting, and join actions
- Maintain widget references for player list, game list, chat display, and buttons
- Provide IP address lookup after user selects a game to join

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GameAvailableMetaserverAnnouncer` | class | Announces a local game to the metaserver for a limited time |
| `GlobalMetaserverChatNotificationAdapter` | class | Receives metaserver notifications and dispatches to UI (chat, room changes) |
| `MetaserverClientUi` | class (abstract) | Base UI controller; concrete type determined at link-time (SDL or Carbon) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `run_network_metaserver_ui()` | function | global | Main entry point; runs metaserver dialog modally and returns selected server IP |
| `setupAndConnectClient()` | function | global | Utility to configure and connect a `MetaserverClient` instance |

## Key Functions / Methods

### run_network_metaserver_ui
- **Signature:** `const IPaddress run_network_metaserver_ui()`
- **Purpose:** Primary UI entry point; displays and runs the metaserver dialog modally
- **Inputs:** None
- **Outputs/Return:** IP address of selected game server
- **Side effects:** Creates and destroys UI; blocks until user cancels or selects a game
- **Calls:** Likely instantiates `MetaserverClientUi::Create()` and calls `GetJoinAddressByRunning()`
- **Notes:** Comment indicates this may eventually disappear or be abstracted further

### setupAndConnectClient
- **Signature:** `void setupAndConnectClient(MetaserverClient& client)`
- **Purpose:** Configure and establish metaserver connection
- **Inputs:** Reference to `MetaserverClient` instance
- **Outputs/Return:** None
- **Side effects:** Modifies client state; initiates network connection
- **Calls:** Not visible (likely delegates to `MetaserverClient` methods)
- **Notes:** Comment marks this as misplaced in header; likely internal utility

### GameAvailableMetaserverAnnouncer::Start
- **Signature:** `void Start(int32 time_limit)`
- **Purpose:** Begin announcing a game on the metaserver
- **Inputs:** `time_limit` ΓÇô duration in seconds for the announcement
- **Outputs/Return:** None
- **Side effects:** Sends game announcement; uses global `gMetaserverClient` (internal state commented)
- **Calls:** Not inferable from this file
- **Notes:** Uses external global client; no per-instance `m_client` member

### MetaserverClientUi::Create
- **Signature:** `static std::auto_ptr<MetaserverClientUi> Create()`
- **Purpose:** Abstract factory; returns platform-specific UI implementation
- **Inputs:** None
- **Outputs/Return:** Auto-pointer to concrete `MetaserverClientUi` subclass
- **Side effects:** Allocates UI instance; implementation determined at link-time
- **Calls:** Not visible (factory resolution at link-time)
- **Notes:** Enables platform abstraction (SDL vs. Carbon)

### MetaserverClientUi::GetJoinAddressByRunning
- **Signature:** `const IPaddress GetJoinAddressByRunning()`
- **Purpose:** Run modal UI and return selected server address
- **Inputs:** None
- **Outputs/Return:** IP address of selected game
- **Side effects:** Runs `Run()` loop; updates `m_joinAddress` on selection
- **Calls:** Calls virtual `Run()` and `Stop()`
- **Notes:** Template method pattern; concrete behavior in subclass `Run()`

**Notes (other methods):**
- Event handlers (`GameSelected`, `PlayerSelected`, `MuteClicked`, `JoinClicked`, `ChatTextEntered`, etc.) dispatch UI interactions to the underlying `MetaserverClient`
- Virtual notification methods (`playersInRoomChanged`, `gamesInRoomChanged`, `receivedChatMessage`) forward metaserver events to widget updates
- `delete_widgets()` cleans up UI resources
- `UpdatePlayerButtons()` and `UpdateGameButtons()` enable/disable controls based on selection state

## Control Flow Notes
**Initialization ΓåÆ UI Loop ΓåÆ Shutdown:**
1. `run_network_metaserver_ui()` or `GetJoinAddressByRunning()` creates/initializes UI
2. User interacts with widgets; event handlers (`ChatTextEntered`, `GameSelected`, etc.) forward actions to `MetaserverClient`
3. `MetaserverClient::pump()` receives server updates; `NotificationAdapter` callbacks update widgets
4. User selects a game and clicks "Join" ΓåÆ `JoinGame()` populates `m_joinAddress`
5. `Run()` returns; caller receives `m_joinAddress`

**Platform Abstraction:**
- `MetaserverClientUi` is abstract; `Create()` factory resolves to platform-specific subclass at link-time
- Widget types (`PlayerListWidget`, `ButtonWidget`, `EditTextWidget`, etc.) are platform-specific but accessed through shared interface
- Included via `shared_widgets.h`, which conditionally includes SDL or Carbon variant

## External Dependencies
- **`network_metaserver.h`** ΓÇô `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`, `MetaserverMaintainedList`
- **`metaserver_messages.h`** ΓÇô `GameListMessage::GameListEntry`, message types
- **`shared_widgets.h`** ΓÇô `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget` base classes
- **Forward declaration** ΓÇô `struct game_info` (defined elsewhere)
- **`IPaddress`** ΓÇô Likely from SDL_net or platform abstraction layer (defined elsewhere)
