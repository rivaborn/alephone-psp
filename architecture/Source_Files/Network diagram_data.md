# Source_Files/Network/ConnectPool.cpp
## File Purpose

Implements a non-blocking TCP connection pool for the Aleph One engine. `NonblockingConnect` wraps individual asynchronous socket connections in SDL threads, while `ConnectPool` is a singleton that manages a fixed-size pool of these connections, allocating on demand and cleaning up finished connections.

## Core Responsibilities

- **NonblockingConnect**: Establish a single TCP connection asynchronously in a dedicated thread; separate DNS resolution from connection to avoid blocking.
- **ConnectPool**: Singleton pool managementΓÇöallocate connection slots on demand, track in-use vs. available slots, and perform lazy cleanup of completed connections.
- **Thread management**: Spawn, track, and join SDL threads for each connection attempt; propagate connection status back to caller.
- **Memory safety**: Use `std::auto_ptr` to ensure cleanup of transient buffers and channels; support explicit abandonment by caller.

## External Dependencies

- **SDL threading:** `SDL_CreateThread`, `SDL_WaitThread` (from `SDL_thread.h`).
- **SDL networking:** `SDLNet_ResolveHost` (from SDL_net library, not explicitly included here but used).
- **CommunicationsChannel:** Defined elsewhere; wraps actual socket I/O.
- **cseries.h:** Likely defines `uint16`, `IPaddress`, and assert macros.
- **Standard library:** `<string>`, `<memory>` (for `std::auto_ptr`).

---

### Potential Issues

- **Memory leak in `fast_free()`:** Slots marked `second=true` are deleted, but the boolean is never reset to true, preventing slot reuse (though they are removed from the list).
- **Destructor disabled:** Cleanup code is commented out, preventing proper resource release at exit.
- **std::auto_ptr deprecated:** This code uses C++98 idiom; modern code would use `std::unique_ptr`.

# Source_Files/Network/ConnectPool.h
## File Purpose
Provides a singleton connection pool manager for non-blocking outbound TCP connections. Manages up to 20 concurrent asynchronous connection attempts, offloading each attempt to an SDL thread to avoid blocking the main game loop.

## Core Responsibilities
- **NonblockingConnect**: Encapsulates a single asynchronous TCP connection attempt with status tracking and thread lifecycle management
- **ConnectPool (singleton)**: Maintains a reusable pool of up to 20 `NonblockingConnect` objects
- Asynchronous connection initiation via separate SDL threads (non-blocking to caller)
- Status polling to check connection progress (Connecting ΓåÆ Connected/Failed states)
- Resource handoff: release a completed `CommunicationsChannel` to the caller for bidirectional communication

## External Dependencies
- **Includes:** `cseries.h` (core types: `uint16`, `IPaddress`), `CommunicationsChannel.h`, `<string>`, `<memory>` (auto_ptr), `<SDL_thread.h>`
- **Defined elsewhere:** `CommunicationsChannel` class (TCPMess module), `IPaddress` type (SDL_net), SDL threading primitives

# Source_Files/Network/Metaserver/metaserver_dialogs.cpp
## File Purpose
Implements the UI dialogs and notification handlers for the metaserver client in Aleph One. Enables players to discover games, browse player lists, participate in chat, and join online games through a central metaserver interface.

## Core Responsibilities
- **Game announcement**: Register and announce games to the metaserver with full game configuration
- **UI orchestration**: Manage modal dialog lifecycle, widget callbacks, and user interactions (game selection, player targeting, chat entry)
- **Chat integration**: Route metaserver chat messages (public, private, broadcast, local) into a unified chat history
- **Player/game list management**: Display and sort available players and games; update UI state based on selection
- **Game joining**: Extract server address/port and signal the main loop with join intent
- **Update checking**: Poll for available updates and notify the user before multiplayer play

## External Dependencies
- **network_metaserver.h**: `MetaserverClient`, `MetaserverPlayerInfo`, `GameListMessage`, `GameDescription`, `NotificationAdapter`
- **metaserver_messages.h**: Message types and utilities
- **preferences.h**: `player_preferences`, `network_preferences`, `environment_preferences` globals
- **network_private.h**: `GAME_PORT` constant
- **alephversion.h**: `A1_DISPLAY_VERSION`, `A1_DISPLAY_PLATFORM` macros
- **SoundManager.h**: `PlayInterfaceButtonSound()` function
- **Update.h**: `Update::instance()`, update status checking
- **progress.h**: `open_progress_dialog()`, `close_progress_dialog()`
- **shared_widgets.h**: Widget classes (`PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`)
- **CSeries / cseries.h**: `pstrdup()`, `a1_p2cstr()`, macro utilities
- **Standard library**: `std::vector`, `std::sort`, `std::string`, `std::auto_ptr`, `boost::bind()`
- **SDL**: `SDL_GetTicks()` for timing

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

## External Dependencies
- **`network_metaserver.h`** ΓÇô `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`, `MetaserverMaintainedList`
- **`metaserver_messages.h`** ΓÇô `GameListMessage::GameListEntry`, message types
- **`shared_widgets.h`** ΓÇô `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget` base classes
- **Forward declaration** ΓÇô `struct game_info` (defined elsewhere)
- **`IPaddress`** ΓÇô Likely from SDL_net or platform abstraction layer (defined elsewhere)

# Source_Files/Network/Metaserver/metaserver_messages.cpp
## File Purpose
Implements serialization/deserialization (deflation/inflation) for metaserver protocol messages in Aleph One's networking layer. Handles encoding/decoding of login credentials, room listings, player info, game descriptions, and chat messages for communication with a central metaserver. Includes platform detection, encryption negotiation, and player color management.

## Core Responsibilities
- Serialize message types to network byte streams (deflate) and deserialize from byte streams (inflate) for transmission
- Provide stream utilities for reading/writing null-terminated strings, padded data, and numeric types
- Implement 13+ message type handlers (login, room management, player lists, game listings, chat, broadcasts)
- Manage player color data with support for custom metaserver color overrides
- Parse and encode game descriptions including scenario IDs, map checksums, physics, and netscript references
- Define static lookup tables for room names, game types, and difficulty levels used in display formatting
- Handle platform detection and encryption scheme negotiation (plaintext, simple, roomserver-based)

## External Dependencies
- **Message framework**: Message.h, SmallMessageHelper base class; CommunicationsChannel.h, MessageDispatcher.h for message routing
- **Stream I/O**: AStream.h (AIStream, AOStream) for byte serialization
- **Game data**: Scenario.h (Scenario::instance() for scenario ID/name/version); map.h (TICKS_PER_SECOND for time calculations); network.h (kNetworkSetupProtocolID)
- **Preferences**: preferences.h (network_preferences, player_preferences); shell.h (_get_player_color, get_player_color)
- **UI/Display**: network_dialogs.h, TextStrings.h (TS_GetCString() for localized game type strings)
- **Utilities**: boost/algorithm/string/predicate.hpp (ends_with); SDL_net.h (IPaddress, SDL_GetTicks)
- **Logging**: Logging.h (logging facilities, conditional on DISABLE_NETWORKING guard)

# Source_Files/Network/Metaserver/metaserver_messages.h
## File Purpose
Defines message types and serializable data structures for metaserver client-server communication. Implements a protocol for login, player/game list sync, chat, and game management using the TCPMess message framework with binary serialization via AIStream/AOStream.

## Core Responsibilities
- Define message type constants for serverΓåöclient bidirectional communication (login, room/player/game lists, chat, authentication)
- Provide serializable message classes inheriting from `SmallMessageHelper` for deflation/inflation with binary streams
- Define aggregate data structures (`GameDescription`, `RoomDescription`, `MetaserverPlayerInfo`) for game state representation
- Support authentication flow (salt exchange, login success, handoff tokens)
- Provide game creation, removal, and state-update messaging
- Support player roster, chat, and private messaging
- Handle scenario/map compatibility checks and game filtering logic

## External Dependencies
- **Message.h:** `SmallMessageHelper`, `DatalessMessage<T>` base classes; `COVARIANT_RETURN` macro
- **AStream.h:** `AIStream`, `AOStream` for binary serialization (abstract; concrete BE/LE variants exist)
- **SDL_net.h:** `IPaddress` type
- **Scenario.h:** `Scenario::instance()` singleton for scenario ID/name/version and compatibility checks
- **network.h:** `kNetworkSetupProtocolID` constant
- **std library:** `<string>`, `<vector>`
- **Boost:** `boost::algorithm::to_lower_copy()` for case-insensitive string handling (imported via using declaration)

# Source_Files/Network/Metaserver/network_metaserver.cpp
## File Purpose

Implementation of the MetaserverClient class, a network client for connecting to the Aleph One metaserver. Handles authentication, room login, player/game list synchronization, chat messaging, game announcement, and player management for multiplayer game discovery and coordination.

## Core Responsibilities

- **Authentication & Connection**: Login to metaserver with encryption, room selection and login, token-based session handling
- **Message Handler Setup**: Inflate/deflate protocol messages, dispatch incoming messages by type to appropriate handlers
- **Player & Game Management**: Maintain synchronized lists of players and games in the current room, support player targeting
- **Chat & Communication**: Process chat messages with built-in commands (.available, .who, .ignore, .games), send private messages, handle broadcasts
- **Game Lifecycle**: Announce games, update player counts, mark games as started/closed/deleted, sync game state with server
- **Ignore List Management**: Toggle player ignore status, filter messages from ignored players, prevent muting guests
- **Connection Lifecycle**: Periodic network pump to send/receive messages, detect disconnection, notify adapter on connection changes

## External Dependencies

- **Networking:** `CommunicationsChannel`, `MessageInflater`, `MessageDispatcher`, `MessageHandler` (TCPMess library)
- **Message Types:** `SaltMessage`, `AcceptMessage`, `DenialMessage`, `ChatMessage`, `PrivateMessage`, `PlayerListMessage`, `GameListMessage`, `RoomListMessage`, etc. (metaserver_messages.h)
- **Game Data:** `GameDescription`, `RoomDescription`, `MetaserverPlayerInfo` (metaserver_messages.h)
- **Utilities:** `Logging.h` (logAnomaly), `boost::algorithm::starts_with`, std containers (set, vector, map)
- **System:** SDL_net (`IPaddress`), standard C++ (string, iostream, algorithm, memory)

# Source_Files/Network/Metaserver/network_metaserver.h
## File Purpose
Defines the metaserver client for Aleph One, a networked game engine. Manages player connections to the metaserver, room selection, chat/messaging, game announcements, and synchronized lists of players and games.

## Core Responsibilities
- Establish and maintain TCP connection to metaserver with authentication
- Synchronize and maintain lists of rooms, players, and games via incremental updates (add/delete/refresh verbs)
- Send and receive chat messages, private messages, and broadcast announcements
- Announce game state changes (creation, start, reset, deletion, player counts)
- Support notification callbacks via NotificationAdapter interface for UI/game integration
- Track local player properties (name, team, away status, targeting) and sync with server
- Manage player ignore lists and game compatibility checking

## External Dependencies

**Notable includes:**
- `metaserver_messages.h` ΓÇö Message type definitions (RoomDescription, PlayerListMessage, GameListMessage, ChatMessage, etc.)
- `<exception>, <vector>, <map>, <memory>, <set>` ΓÇö STL containers and memory management
- `Logging.h` ΓÇö Logging macros (logAnomaly1, etc.)

**Forward declarations / defined elsewhere:**
- CommunicationsChannel ΓÇö TCP communication abstraction
- MessageInflater ΓÇö Message deserialization/decompression
- MessageDispatcher, MessageHandler ΓÇö Message routing and handling base classes
- Message, ChatMessage, PrivateMessage, BroadcastMessage ΓÇö Message types (defined in metaserver_messages.h)

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

# Source_Files/Network/network.cpp
## File Purpose

Core multiplayer networking implementation for Aleph One game engine. Manages network topology, player joining/dropping, game state transitions, and distribution of map/game data across the network. Supports both Ring and Star game protocols with message-based communication using SDL_net and TCP.

## Core Responsibilities

- **Network initialization & teardown**: Set up DDP socket, topology structures, establish connections
- **Player lifecycle management**: Handle joining players, dropping disconnected players, topology updates
- **Game protocol coordination**: Delegate to RingGameProtocol or StarGameProtocol for frame/action handling
- **Map & game data distribution**: Stream map/physics/Lua script data from gatherer to joiners
- **Message routing**: Dispatch incoming network messages (chat, capabilities, topology, etc.) via MessageDispatcher
- **Join state machine**: Manage connecting ΓåÆ joining ΓåÆ waiting ΓåÆ active phases for joiners
- **Topology maintenance**: Track player positions, addresses, identifiers; propagate changes to all players
- **Feature negotiation**: Validate player capabilities (protocol support, compression, Lua, etc.)
- **Ignore lists & player muting**: Per-local-player ignore/mute management for chat and microphone

## External Dependencies

- **SDL_net**: TCP/UDP socket abstraction (TCPsocket, UDPsocket, SDLNet_SocketSet)
- **Message classes**: JoinerInfoMessage, CapabilitiesMessage, AcceptJoinMessage, TopologyMessage, etc. (defined in network_messages.h)
- **CommunicationsChannel**: TCP message framing and buffering (defined elsewhere)
- **NetworkGameProtocol, RingGameProtocol, StarGameProtocol**: Protocol state machines
- **ConsoleCallbacks, GatherCallbacks, ChatCallbacks**: Application-level feedback hooks
- **MetaserverClient**: Metaserver registration and updates
- **Logging**: logAnomaly(), logError(), logNote() for anomaly/debug reporting
- **Progress UI**: open_progress_dialog(), set_progress_dialog_message(), draw_progress_bar(), close_progress_dialog()
- **Preferences**: network_preferences, player_preferences, environment_preferences (global)
- **map.h**: entry_point, TICKS_PER_SECOND, get_map_for_net_transfer(), process_net_map_data()
- **game_errors.h, game_data/game_info**: Game state structures (opaque void* in this file)
- **Lua**: LoadLuaScript() for netscript loading
- **ConnectPool**: Nonblocking TCP connection pooling with retry/resolution

**Defined elsewhere but called here:**
- NetSync(), NetUnSync() ΓÇô synchronization primitives
- NetGetLocalPlayerIndex(), NetGetPlayerIdentifier(), NetGetNumberOfPlayers() ΓÇô query topology
- NetGetPlayerData(), NetGetGameData() ΓÇô query game state
- process_action_flags() ΓÇô dispatch action flags from packets to game logic
- screen_printf(), alert_user() ΓÇô UI feedback
- machine_tick_count() ΓÇô timing

# Source_Files/Network/network.h
## File Purpose
Public interface header for the Aleph One game engine's network subsystem. Defines types, callbacks, and function prototypes for multiplayer game gathering, joining, state synchronization, and in-game message distribution across network players.

## Core Responsibilities
- Define game and player information structures passed over network
- Manage network state machine (gathering ΓåÆ joining ΓåÆ waiting ΓåÆ active ΓåÆ shutdown)
- Provide game gathering (host side) and joining (client side) mechanisms
- Handle player topology setup and synchronization
- Distribute game data, updates, and chat messages across network
- Manage player connections, disconnections, and color/team changes
- Measure and track network latency for prediction
- Enforce cheat restrictions based on network mode

## External Dependencies
- **cseries.h, cstypes.h**: Portability macros and cross-platform types (int16, uint16, int32, uint32, byte, OSErr).
- **C++ STL**: `std::string` used in chat callbacks.
- **Forward references** (defined elsewhere): `entry_point`, `player_start_data`, `SSLP_ServiceInstance`.
- **Callback interface pattern**: GatherCallbacks, ChatCallbacks established but invoked by network.c (not shown).

# Source_Files/Network/network_audio_shared.h
## File Purpose
Header defining shared data structures and constants for network audio (VoIP) functionality in Marathon: Aleph One. Provides a unified format specification between the network microphone capture and speaker playback code.

## Core Responsibilities
- Define the in-memory header structure for network audio packets
- Declare audio format flags (e.g., team-restricted distribution)
- Specify and document the current network audio encoding parameters (sample rate, bit depth, channels)
- Centralize audio format constants to ensure consistency across microphone and speaker implementations

## External Dependencies
- `cseries.h` ΓÇö provides `uint32` type and platform abstraction macros

**Notes**: 
- The comment indicates microphone code currently hardcodes 11025 Hz unsigned 8-bit mono, conflicting with the stated 8000 Hz / 16-bit constants. Format negotiation/versioning via the reserved field or flags enum is suggested but not yet implemented.
- File authored Feb 1, 2003, extracted from `network_speaker_sdl.h`.

# Source_Files/Network/network_capabilities.cpp
## File Purpose
Defines static string constants for the Capabilities versioning system, used as keys to store and query network protocol and feature support versions during multiplayer game sessions.

## Core Responsibilities
- Initialize static string constant keys for capability lookup in the Capabilities map
- Provide named constants for protocol and feature versioning (Gameworld, Star, Ring, Lua, Speex, Gatherable, ZippedData)

## External Dependencies
- `#include "network_capabilities.h"` ΓÇö defines Capabilities class and version constants
- `cseries.h` (via header) ΓÇö standard definitions
- `<string>`, `<map>` (standard library) ΓÇö used in Capabilities typedef

# Source_Files/Network/network_capabilities.h
## File Purpose

Defines a versioning system for network capabilities and game engine features in Aleph One. The `Capabilities` class wraps a map of feature names to version numbers, enabling peers to negotiate protocol compatibility during network setup (gathering, joining, or protocol exchanges).

## Core Responsibilities

- Define version constants for core network protocols (Gameworld PRNG/physics, Star, Ring)
- Define version constants for optional engine features (Lua scripting, Speex audio, zipped data support)
- Provide a type-safe key-value store (`Capabilities` class) for capability versioning
- Enforce a maximum key size limit (1024 bytes) to prevent buffer issues
- Enable capability negotiation by exporting standardized capability names

## External Dependencies

- **cseries.h**: Provides platform/compiler abstractions and common types (e.g., `uint32`)
- **\<string\>**: Standard C++ string class
- **\<map\>**: Standard C++ map container

# Source_Files/Network/network_data_formats.cpp
## File Purpose
Implements bidirectional serialization functions (netcpy overloads) that convert between platform-native and network-portable binary formats. Handles byte-order swapping and packing/unpacking of network packet structures for the Aleph One game engine's multiplayer layer, ensuring consistent data representation across different platforms and architectures.

## Core Responsibilities
- Convert NetPacketHeader between native and network byte order
- Convert NetPacket (ring buffer packets) between native and network formats, handling player action flags
- Swap byte order for uint32 arrays on little-endian platforms
- Convert NetDistributionPacket (broadcast packets) between formats
- Convert IPaddress (host/port pairs) to/from network byte order without swapping
- Convert network_audio_header structures between formats
- Ensure portable network serialization via stream-based packing/unpacking

## External Dependencies
- **Packing.h**: Provides `ValueToStream()`, `StreamToValue()`, `ListToStream()` for byte-order-aware serialization
- **network_data_formats.h**: Declarations of all netcpy overloads and _NET struct definitions
- **network.h, network_private.h**: Definitions of native network structures (NetPacketHeader, NetPacket, etc.)
- **network_audio_shared.h**: Definition of network_audio_header
- **cseries.h**: Provides ALEPHONE_LITTLE_ENDIAN macro and stdint types (uint8, uint16, uint32, int32, int16)

# Source_Files/Network/network_data_formats.h
## File Purpose
Defines platform-independent network packet structures and serialization/deserialization functions. Provides a conversion layer between native C structs (used internally) and fixed-format "_NET" wire structures (transmitted over network), ensuring consistent byte ordering and padding across all platforms.

## Core Responsibilities
- Define fixed-size wire-format packet structures (`*_NET` variants) with byte-array representation
- Declare `netcpy()` conversion functions for bidirectional marshaling between native and wire formats
- Handle endianness conversions (byte-swapping on little-endian architectures)
- Ensure consistent network protocol serialization across platforms (Windows, Mac, Linux)

## External Dependencies
- **cseries.h** ΓÇö provides `ALEPHONE_LITTLE_ENDIAN` conditional definition; includes SDL byte-order macros
- **network.h** ΓÇö public API; defines `MAXIMUM_NUMBER_OF_NETWORK_PLAYERS` constant
- **network_private.h** ΓÇö defines native struct types (`NetPacketHeader`, `NetPacket`, `NetDistributionPacket`, `IPaddress`)
- **network_audio_shared.h** ΓÇö defines `network_audio_header` native struct
- **cstring.h** (implicit) ΓÇö `memcpy()` for big-endian fast path in uint32 overload

# Source_Files/Network/network_dialog_widgets_sdl.cpp
## File Purpose
Implements network-related dialog widgets for the SDL UI system in Aleph One (Marathon engine). Provides custom widgets for displaying discovered network players, in-game player status with score/carnage graphs, and level selection for network games. Serves both the gather/join phase and postgame carnage report display.

## Core Responsibilities
- Display and manage lists of players discovered on the network
- Render player icons, names, and score/kill/death bars during and after games
- Handle mouse interactions with player elements (selection, graph clicks)
- Layout player names and bars to avoid overlapping text
- Provide level/entry-point selection for network games
- Convert between network topology data and display-ready player entries

## External Dependencies
- **SDL graphics:** `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`, event types
- **Network:** `network.h` (topology, `NetGetNumberOfPlayers()`, `NetGetPlayerData()`), `SSLP_API.h` (service discovery), `prospective_joiner_info`
- **Game world:** `player.h` (player data, `MAXIMUM_NUMBER_OF_PLAYERS`, `NUMBER_OF_TEAM_COLORS`, `get_player_data()`), `player_info`, `entry_point`
- **UI framework:** `sdl_widgets.h` (`w_list<>`, `w_select_button`, widget base class), `dialog` class
- **Rendering:** `screen_drawing.h` (`draw_text()`, `text_width()`, theme colors), `sdl_fonts.h` (`font_info`), `HUDRenderer.h` (indirectly for color definitions)
- **Player visuals:** `PlayerImage_sdl.h` (`PlayerImage` class)
- **Text layout:** `TextLayoutHelper` (avoid overlapping text)
- **Strings:** `TextStrings.h` (`TS_GetCString()`), `network_dialogs.h` (`net_rank` structure)
- **Utilities:** `collection_definition.h`, `preferences.h`, `shell.h` (global sound/interface functions)

**Defined elsewhere:** `get_dialog_player_color()`, `calculate_ranking_text_for_post_game()`, `calculate_max_kills()`, `get_net_color()`, `dialog_cancel` callback, `clear_screen()`, `DIALOG_CLICK_SOUND`, etc.

# Source_Files/Network/network_dialog_widgets_sdl.h
## File Purpose
Defines custom SDL widgets for network-related dialogs in Aleph One's networking UI. Provides specialized list widgets for discovering/displaying network players, a postgame carnage report display widget, and a level selector that adapts to game type constraints.

## Core Responsibilities
- **Player discovery list**: Manages SSLP callback integration to display available network players dynamically
- **Player roster display**: Shows players currently in a game for both gather/join phases and postgame carnage reports
- **Postgame graph rendering**: Renders kill/score bars, player icons, and statistics in multiple visual layouts
- **Level/entry point selection**: Validates and presents level options compatible with the current game type
- **Callback management**: Routes user interactions (clicks, selections) back to dialog code via function pointers

## External Dependencies
- **Includes**: `sdl_widgets.h` (base widget classes), `SSLP_API.h` (service discovery), `player.h` (player constants like `MAXIMUM_PLAYER_NAME_LENGTH`), `PlayerImage_sdl.h` (player rendering), `network_dialogs.h` (ranking structs, game type info)
- **External symbols**: `prospective_joiner_info` (network.h), `entry_point` (map.h), `net_rank` (network_dialogs.h), `TextLayoutHelper` (not defined here), `bar_info` (not defined here)
- **Base classes**: `w_list<T>` (template from sdl_widgets.h), `w_select_button` (sdl_widgets.h), `widget` (sdl_widgets.h)

# Source_Files/Network/network_dialogs.cpp
## File Purpose

Implements cross-platform (SDL/macOS) UI dialogs for network game setup. Handles gathering (hosting), joining, and configuring multiplayer games, with integration for metaserver advertising, LAN discovery via SSLP, and in-game/pre-game chat.

## Core Responsibilities

- **Network Gather Dialog**: UI for hosting a game, accepting joining players, starting game
- **Network Join Dialog**: UI for finding and connecting to gathering games via LAN or metaserver
- **Game Setup Dialog**: Configure game options (map, difficulty, game type, time/kill limits, player appearance)
- **Metaserver Integration**: Advertise hosted games and discover Internet games via metaserver
- **LAN Discovery (SSLP)**: Find/announce games on local network using kNetworkSetupProtocolID
- **Chat System**: Pre-game chat between gathered/joining players and metaserver chat
- **Progress Dialogs**: Display loading/setup progress during game initialization

## External Dependencies

- **Includes:** `network.h`, `network_private.h`, `SSLP_API.h`, `metaserver_dialogs.h`, `TextStrings.h`, `network_dialog_widgets_sdl.h`, `SoundManager.h`, `progress.h`
- **Network layer** (defined elsewhere): `NetEnter()`, `NetGather()`, `NetGameJoin()`, `NetCheckForNewJoiner()`, `NetUpdateJoinState()`, `NetGetGameData()`, `NetGetNumberOfPlayers()`, `NetChangeColors()`, `NetSetGatherCallbacks()`, `NetSetChatCallbacks()`, `SendChatMessage()`
- **Metaserver**: `MetaserverClient` class (singleton, manages chat & game advertising)
- **LAN discovery (SSLP)**: `SSLP_Allow_Service_Discovery()`, `SSLP_Disallow_Service_Discovery()`, `SSLP_Pump()`, `SSLP_Locate_Service_Instances()`, `SSLP_Stop_Locating_Service_Instances()` (external C functions)
- **Preferences**: `network_preferences`, `player_preferences`, `serial_preferences` (global pointers)
- **UI/Dialogs (SDL)**: `dialog`, `w_button`, `w_toggle`, `w_text_entry`, `w_select_popup`, `vertical_placer`, `horizontal_placer`, `table_placer`, `tab_placer` (widget framework, defined elsewhere)
- **Widget adapters**: `ButtonWidget`, `EditTextWidget`, `ToggleWidget`, `PopupSelectorWidget`, `ColorfulChatWidget`, `JoiningPlayerListWidget`, `PlayersInGameWidget`, etc. (defined in `network_dialog_widgets_sdl.h`)

# Source_Files/Network/network_dialogs.h
## File Purpose

Defines abstract dialog classes and interfaces for network game initialization (gathering players, joining, and setup), post-game statistics display, and metaserver integration. Provides a platform-agnostic interface implemented separately for Carbon (Mac) and SDL platforms.

## Core Responsibilities

- Define abstract factory classes for game dialogs (gather, join, setup) with platform-specific implementations
- Manage callback interfaces for network events during pre-game and chat phases
- Define data structures for ranking, game outcome tracking, and UI state
- Expose postgame carnage report rendering functions (graphs, rankings, damage stats)
- Provide enums and constants for dialog items, string resources, and game configuration limits

## External Dependencies

- **Notable includes:**  
  `player.h` (MAXIMUM_NUMBER_OF_PLAYERS), `network.h` (game_info, player_info, prospective_joiner_info, ChatCallbacks, GatherCallbacks), `network_private.h` (JoinerSeekingGathererAnnouncer), `FileHandler.h` (FileSpecifier, FileChooserWidget), `network_metaserver.h` (metaserver client), `metaserver_dialogs.h` (GlobalMetaserverChatNotificationAdapter), `shared_widgets.h` (widget base classes: ButtonWidget, EditTextWidget, SelectorWidget, ToggleWidget, PlayersInGameWidget, etc.)

- **STL:** `<map>` (player tracking), `<set>` (game protocol), `<string>`, `<memory>` (auto_ptr)

- **Defined elsewhere:** `prospective_joiner_info`, `player_info`, `game_info` (from network.h), CFStringRef and OSType (Carbon/Mac specifics for NIB-based dialogs)

# Source_Files/Network/network_distribution_types.h
## File Purpose
Defines enumeration constants for network audio distribution type IDs in the Aleph One game engine. Centralizes distribution type identifiers to avoid conflicts across the codebase and maintains backward compatibility with older versions.

## Core Responsibilities
- Define network audio distribution type identifiers for use in `NetDistributeInformation` and `NetAddDistributionFunction`
- Maintain a registry for legacy vs. modern audio streaming protocols
- Provide a centralized, conflict-free location for distribution type constants

## External Dependencies
- Standard C header guards (`#ifndef`, `#define`, `#endif`)
- No external includes or symbol dependencies

# Source_Files/Network/network_dummy.cpp
## File Purpose
Provides stub/dummy implementations of all network interface functions for single-player or network-disabled builds. All functions return trivial values or perform no operations, allowing the game to compile and run without network support. This is a fallback implementation that satisfies the linker when actual network code is unavailable.

## Core Responsibilities
- Provide no-op implementations of all network API functions declared in `network.h`
- Enable compilation when real network subsystem is disabled or unavailable
- Allow single-player game operation by returning safe default values
- Stub all player query, synchronization, and map change functions
- Disable network-dependent features (crosshair, tunnel vision, behind-view in multiplayer)

## External Dependencies
- **Includes**: `cseries.h` (base types, compiler macros), `map.h` (struct `entry_point`), `network.h` (function declarations), `network_games.h` (presumably for game-mode queries)
- **External symbols**: All function names declared in `network.h`; `entry_point` struct defined in `map.h`

# Source_Files/Network/network_games.cpp
## File Purpose
Implements multiplayer network game mode logic for the Aleph One engine. Manages game-specific rules, scoring mechanics, player rankings, compass navigation targets, and game-over conditions across nine distinct netgame types (CTF, King of the Hill, tag, rugby, etc.).

## Core Responsibilities
- **Game initialization** per netgame type (beacon placement, state setup)
- **Per-frame game rule updates** (scoring, ball/flag handling, time tracking)
- **Player and team ranking calculations** with type-specific scoring formulas
- **Game-over condition evaluation** (time/kill/capture limits, Lua overrides)
- **Compass navigation** (directional indicators to objectives/targets)
- **Player event handling** (kill consequences, team assignment)
- **UI text generation** (ranking displays, game mode labels, post-game summaries)

## External Dependencies
- **cseries.h** ΓÇö base utilities (assert, sprintf, csprintf, vhalt)
- **map.h** ΓÇö map geometry (polygon_data, map_polygons), game_data, dynamic_data, macros (GET_GAME_TYPE, GET_GAME_OPTIONS, GET_GAME_PARAMETER)
- **items.h** ΓÇö item constants (BALL_ITEM_BASE), find_player_ball_color()
- **lua_script.h** ΓÇö GetLuaScoringMode(), GetLuaGameEndCondition(), lua compass control
- **player.h** ΓÇö player_data, damage_record, PLAYER_IS_DEAD, PLAYER_IS_TOTALLY_DEAD macros
- **network.h** ΓÇö network subsystem declarations
- **network_games.h** ΓÇö own header (function prototypes, compass enums)
- **game_window.h** ΓÇö mark_player_network_stats_as_dirty()
- **SoundManager.h** ΓÇö (included but unused in visible code)

**Defined-elsewhere symbols**: `dynamic_world`, `current_player_index`, `current_player`, `temporary`, `map_polygons`, `get_player_data()`, `get_polygon_data()`, `get_object_data()`, `destroy_players_ball()`, `arctangent()`, `NORMALIZE_ANGLE()`, `PLAYER_IS_DEAD()`, `PLAYER_IS_TOTALLY_DEAD()`

# Source_Files/Network/network_games.h
## File Purpose
Declares the network game management interface for multiplayer games. Provides functions to initialize and update network game state, calculate player/team rankings, query game conditions (over/balls/scores), and retrieve UI text for rankings and network compass navigation.

## Core Responsibilities
- Initialize and update networked multiplayer game state each frame
- Calculate and rank players/teams by kills/deaths
- Format ranking data for UI display (in-game and post-game)
- Query game mode capabilities (has scores, has balls/flags)
- Track player-versus-player kills
- Determine game-over conditions
- Provide network compass state for each player

## External Dependencies
- `NUMBER_OF_TEAM_COLORS` ΓÇö constant defined elsewhere (likely game mode / balance config)
- No explicit #include directives visible, suggesting minimal local coupling

# Source_Files/Network/network_lookup_sdl.cpp
## File Purpose
SDL-based implementation of network service discovery and registration for Aleph One using the SSLP (Simple Service Location Protocol). Bridges the game's networking layer with cross-platform SSLP for discovering available game servers and advertising the local player to other clients.

## Core Responsibilities
- Build SSLP-compatible service type strings (combining service type + version)
- Start/stop searching for available network services via callbacks
- Register and unregister the local player/service instance for discovery
- Maintain lookup and registration state (two file-static bools)
- Support address hinting for inter-domain service discovery
- Convert between Pascal strings (P-strings) and C-style null-terminated strings

## External Dependencies
- `SSLP_API.h`: `SSLP_ServiceInstance`, `SSLP_Locate_Service_Instances`, `SSLP_Stop_Locating_Service_Instances`, `SSLP_Allow_Service_Discovery`, `SSLP_Hint_Service_Discovery`, `SSLP_Disallow_Service_Discovery`, constants `SSLP_MAX_TYPE_LENGTH`, `SSLP_MAX_NAME_LENGTH`
- `SDL_net`: `IPaddress`, `SDLNet_ResolveHost`
- `sdl_network.h`: `NetAddrBlock`, `NetEntityName`, `OSErr`
- Standard C: `memcpy`, `sprintf`, `strlen`, `strdup`, `free`

# Source_Files/Network/network_lookup_sdl.h
## File Purpose
SDL-based header providing network service discovery and registration functions for multiplayer games. Acts as a wrapper around the SSLP (Simple Service Location Protocol) to enable game instances to register themselves and discover other players on local networks, abstracting platform-specific network discovery (replacing MacOS AppleTalk NBP).

## Core Responsibilities
- Register local game service instances for network discovery
- Unregister service instances when shutting down
- Initiate discovery of remote game service instances via callbacks
- Manage service lookup lifecycle (open/close operations)
- Bridge Aleph One's network interface with SDL-based SSLP implementation

## External Dependencies
- **SSLP_API.h** ΓÇô Provides `SSLP_ServiceInstance`, callback typedef, and low-level SSLP functions (`SSLP_Pump()`, `SSLP_Locate_Service_Instances()`, `SSLP_Allow_Service_Discovery()`, etc.).
- **SDL_net** (via SSLP_API.h) ΓÇô Cross-platform UDP/IP networking.
- **NETWORK_NAMES.C** ΓÇô Implementation of registration functions.
- **NETWORK_LOOKUP.C** ΓÇô Implementation of discovery/lookup functions.
- **Aleph One core** ΓÇô Defines `OSErr` and Pstring conventions.

# Source_Files/Network/network_messages.cpp
## File Purpose
Implements serialization and deserialization (inflate/deflate) for all network message types used during game setup and multiplayer communication. Provides helper utilities for string packing and binary encoding, plus zlib-based compression for large data chunks (maps, physics, Lua scripts).

## Core Responsibilities
- Serialize/deserialize network message objects to/from binary streams
- Convert between C strings, Pascal strings, and stream format
- Handle endianness via `AIStreamBE`/`AOStreamBE` (big-endian) stream classes
- Compress/decompress large data payloads (maps, physics, Lua) using zlib
- Maintain network protocol compatibility by preserving struct field order and sizes
- Support message types: hello, joiner info, capabilities, topology, chat, colors, warnings, join acceptance, client info

## External Dependencies
- **cseries.h** ΓÇö base platform/compiler compatibility layer
- **AStream.h** ΓÇö `AIStream`, `AOStream`, `AIStreamBE`, `AOStreamBE` classes for endian-aware serialization
- **network_messages.h** ΓÇö message class declarations (base classes like `SmallMessageHelper`, `BigChunkOfDataMessage`)
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `ClientChatInfo`, protocol constants
- **network_data_formats.h** ΓÇö network-safe struct packing/unpacking utilities
- **zlib.h** ΓÇö `compress()`, `uncompress()` for data compression
- **Message.h** (assumed) ΓÇö `UninflatedMessage` base class and message framework

# Source_Files/Network/network_messages.h
## File Purpose
Defines network message types and classes used for Aleph One multiplayer game setup and initialization. Implements serializable message classes that communicate player info, capabilities, topology, map/physics/script data, and chat over the network using a stream-based encoding/decoding pattern.

## Core Responsibilities
- Define message type IDs for all network protocol messages (kHELLO_MESSAGE, kJOINER_INFO_MESSAGE, etc.)
- Implement typed message classes that inherit from SmallMessageHelper or BigChunkOfDataMessage base classes
- Provide serialization (deflate) and deserialization (inflate) via AIStream/AOStream
- Support compressed variants of large data messages (zipped map, physics, Lua)
- Manage client state machine and message dispatch handlers during game gather/join
- Provide template utilities for creating strongly-typed message classes

## External Dependencies
- **AStream.h** ΓÇö AIStream, AOStream (big/little-endian serialization)
- **Message.h** ΓÇö SmallMessageHelper, BigChunkOfDataMessage, SimpleMessage, DatalessMessage, UninflatedMessage, Message base class
- **network_capabilities.h** ΓÇö Capabilities class (map of string ΓåÆ uint32 feature versions)
- **network_private.h** ΓÇö NetPlayer, NetTopology, ClientChatInfo, CommunicationsChannel, MessageDispatcher, MessageHandler, prospective_joiner_info
- **SDL_net.h** ΓÇö Uint8, SDL network types
- **cseries.h** ΓÇö Standard library includes, int16/uint32 typedefs

# Source_Files/Network/network_microphone_coreaudio.cpp
## File Purpose
Implements audio capture from the system microphone on macOS using CoreAudio. Captures raw audio samples from the default input device, buffers them, and forwards to a network transmission pipeline. Conditionally includes Speex compression support.

## Core Responsibilities
- Initialize CoreAudio HAL (Hardware Abstraction Layer) for audio input
- Enumerate sample rates and configure the input device to a compatible rate
- Set up audio stream format conversion (to mono, 16-bit PCM)
- Allocate and manage audio buffers for captured frames
- Implement input callback to receive audio data from the hardware
- Buffer incoming samples and batch-send when a packet threshold is reached
- Control microphone state (start/stop capture)
- Release all CoreAudio resources on shutdown

## External Dependencies
- **Includes:** `<Carbon/Carbon.h>` (macOS CoreAudio types), `<AudioUnit/AudioUnit.h>` (AudioUnit API).
- **Defined elsewhere:** `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()` (from network_microphone_shared.h), `init_speex_encoder()`, `destroy_speex_encoder()` (ifdef SPEEX, implementation location unknown).

# Source_Files/Network/network_microphone_sdl_alsa.cpp
## File Purpose
Implements network microphone audio capture on Linux using ALSA (Advanced Linux Sound Architecture). Provides initialization, hardware/software configuration, and async callback-driven frame capture with optional Speex compression support. Conditionally compiled only when `HAVE_ALSA` is defined; falls back to dummy implementation otherwise.

## Core Responsibilities
- Initialize ALSA PCM device and configure for 8kHz mono 16-bit capture
- Allocate and set hardware/software parameters (access mode, format, sample rate, channels, period size)
- Open and close ALSA capture device handle
- Register/unregister async callback handler for frame-ready events
- Control microphone capture state (start/stop)
- Read available audio frames and queue for network transmission
- Handle buffer underrun recovery and optional Speex encoder initialization
- Report microphone availability on this platform

## External Dependencies
- `<alsa/asoundlib.h>` ΓÇö ALSA PCM and async APIs (Linux only)
- `"cseries.h"` ΓÇö Common types (`uint8`, `OSErr`), endianness macro (`ALEPHONE_LITTLE_ENDIAN`)
- `"network_speex.h"` (conditional `SPEEX`) ΓÇö Speex encoder/decoder
- `"preferences.h"` ΓÇö Global `network_preferences` struct
- `"network_microphone_shared.h"` ΓÇö Shared interface: `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- `"network_microphone_sdl_dummy.cpp"` ΓÇö Fallback dummy implementation (included at end if `HAVE_ALSA` undefined)

# Source_Files/Network/network_microphone_sdl_dummy.cpp
## File Purpose
Provides stub implementations of SDL-style network microphone functions for the Marathon: Aleph One game engine. This dummy module satisfies the linker and permits graceful degradation on platforms without microphone support, returning `false` from availability checks to indicate the feature is not implemented.

## Core Responsibilities
- Define empty stubs for microphone lifecycle functions (`open`/`close`)
- Provide state control stub (`set_network_microphone_state`)
- Advertise non-implementation via `is_network_microphone_implemented()` returning `false`
- Supply an idle hook (`network_microphone_idle_proc`) for the main loop to call without crashing

## External Dependencies
- No includes or external symbols.

# Source_Files/Network/network_microphone_sdl_win32.cpp
## File Purpose
Windows-specific implementation of network microphone capture using DirectX DirectSound API. Manages audio input hardware initialization, circular buffer capture, and transmission of frames over the network for voice communication in multiplayer games.

## Core Responsibilities
- Initialize and manage DirectSound capture buffers and COM interfaces
- Query hardware capabilities and select optimal audio format from preferences
- Implement circular buffer capture and read position tracking
- Transmit captured audio frames to network layer (packeted)
- Manage microphone lifecycle: open, start/stop transmission, close
- Interface with optional Speex audio compression codec
- Provide periodic idle polling for frame transmission

## External Dependencies
- **DirectX SDK:** `<dsound.h>` ΓÇô DirectSoundCapture, DirectSoundCaptureBuffer COM interfaces; DSCBUFFERDESC, WAVEFORMATEX structures
- **Cross-platform layer:** `network_microphone_shared.h` ΓÇô `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- **Platform utilities:** `cseries.h` ΓÇô type definitions, macros
- **Logging:** `Logging.h` ΓÇô `logContext()`, `logAnomaly()`, `logAnomaly1()`, `logAnomaly3()`
- **Preferences:** `preferences.h` ΓÇô `network_preferences` global (Speex encoder flag)
- **Audio compression (optional):** `network_speex.h` ΓÇô `init_speex_encoder()`, `destroy_speex_encoder()` (compiled only if `SPEEX` defined)
- **Speaker layer:** `network_speaker_sdl.h` ΓÇô included but not directly used in this file

# Source_Files/Network/network_microphone_shared.cpp
## File Purpose
Utility code for capturing and encoding network microphone audio. Handles format conversion, optional Speex compression, sample-rate interpolation, and packet distribution for real-time voice over the network in Aleph One multiplayer games.

## Core Responsibilities
- Announce and track microphone capture format (sample rate, channels, bit depth)
- Extract audio samples from raw capture buffers with format conversion (mono/stereo, 8/16-bit)
- Perform sample-rate conversion via fixed-point interpolation
- Encode audio frames using Speex codec (if enabled at compile-time)
- Accumulate encoded frames into network packets
- Distribute audio packets to the game network layer

## External Dependencies
- **Includes:** `cseries.h` (base types, macros), `network_speaker_sdl.h`, `network_data_formats.h` (struct serialization), `network_distribution_types.h` (distribution IDs), `network_speex.h` (encoder globals), `map.h` (game options, `GET_GAME_OPTIONS()`).
- **Speex library** (conditional `SPEEX`): `speex/speex.h`.
- **Symbols defined elsewhere:** `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame`, `kNetworkAudioSamplesPerPacket` (constants); `gEncoderState`, `gEncoderBits` (Speex encoder); `NetDistributeInformation`, `netcpy` (network layer); `received_network_audio_proc` (test loopback); `GET_GAME_OPTIONS()`, `_force_unique_teams` (game state).

# Source_Files/Network/network_microphone_shared.h
## File Purpose
Defines the shared interface for network microphone implementations in Marathon: Aleph One. Provides three key functions for announcing audio capture format, buffering and sending audio data, and querying packet size requirements. Internal use only by netmic implementation files.

## Core Responsibilities
- Announce microphone capture format (sample rate, stereo/mono, bit depth) and track it internally
- Buffer captured audio data and determine when sufficient data exists to send a network packet
- Send audio data over the network, respecting format and buffering constraints
- Provide queries for packet size calculations to guide preprocessing decisions

## External Dependencies
- Platform integer types: `uint32`, `uint8`, `int32` (defined elsewhere)
- Callers are netmic implementation files; function implementations are defined elsewhere in the network module

# Source_Files/Network/network_private.h
## File Purpose
Private header defining internal structures, constants, and interfaces for the network subsystem. Contains packet definitions for ring-protocol communication, game topology structures, error codes, and service discovery management. Not intended for use outside the networking module.

## Core Responsibilities
- Define network packet structures (ring packets, distribution packets, player topology)
- Specify packet tags, types, and protocol constants for ring-protocol gameplay
- Enumerate error codes and network states for internal error handling
- Define game-specific data structures (gathering, chat, game info, player info)
- Provide service discovery interface classes (SSLP-based gatherer/joiner announcement)
- Supply distribution info lookup for custom game data types

## External Dependencies

- **cstypes.h** ΓÇö Fixed-width integer types (int8, int16, uint32, etc.), NONE constant
- **sdl_network.h** ΓÇö SDL_net types (IPaddress, UDPsocket, TCPsocket); game-port UDP transport
- **network.h** ΓÇö Public API; game_info, player_info, NetDistributionProc typedef, ChatCallbacks, GatherCallbacks
- **SSLP_API.h** ΓÇö Service discovery: SSLP_ServiceInstance, SSLP_Pump, SSLP_Allow_Service_Discovery, SSLP_Locate_Service_Instances
- **\<memory\>** ΓÇö std::string for ClientChatInfo
- **Forward declarations** ΓÇö CommunicationsChannel, MessageDispatcher, MessageHandler, Message (used in implementation, not defined here)

# Source_Files/Network/network_sound.h
## File Purpose
Header file defining the main interface for network audio functionality (microphone input and speaker output) in the Aleph One engine. Provides unified SDL/Mac-compatible abstractions for capturing and playing back networked voice communication between game clients.

## Core Responsibilities
- Speaker system initialization, idle processing, and shutdown
- Queueing and playback of received network audio data
- Player microphone muting controls
- Microphone capability detection and activation
- Microphone input processing via idle callbacks
- Platform abstraction (SDL/Mac audio interfaces unified under this API)

## External Dependencies
- **cseries.h**: Platform abstraction layer; provides OSErr type, SDL integration, and basic type definitions
- **NETWORK_SPEAKER.C**: Implementation of speaker system functions
- **NETWORK_MICROPHONE.C**: Implementation of microphone system functions

# Source_Files/Network/network_speaker_sdl.cpp
## File Purpose
Implements network audio playback for SDL platforms in the Aleph One game engine. Manages incoming network audio buffers, generates noise for buffering smoothness, and coordinates with the audio mixer for real-time playback during networked games.

## Core Responsibilities
- Allocate and manage audio data buffer pools for incoming network audio
- Generate and queue noise buffers to smooth audio playback during buffering
- Coordinate buffer lifecycle between network receiver and audio mixer threads
- Track speaker state (active/inactive) and handle dry-queue edge cases
- Initialize Speex decoder support when enabled
- Clean up resources on shutdown

## External Dependencies
- **network_sound.h, network_speaker_sdl.h** ΓÇö Interface definitions for speaker API.
- **network_distribution_types.h** ΓÇö Network distribution type constants (unused in this file).
- **CircularQueue.h** ΓÇö Template-based queue container.
- **world.h** ΓÇö `local_random()` for noise generation.
- **Mixer.h** ΓÇö `Mixer::instance()` singleton for audio system integration.
- **network_speex.h** (conditional) ΓÇö Speex codec decoder initialization/destruction when `SPEEX` defined.

# Source_Files/Network/network_speaker_sdl.h
## File Purpose
Defines the interface between SDL network audio receiving code and SDL sound playback routines for real-time microphone playback in networked Marathon: Aleph One games. Acts as a bridge layer abstracting buffer management between the network speaker subsystem and audio output.

## Core Responsibilities
- Define buffer descriptor structure for network audio data transmission
- Declare flag enumeration for buffer lifecycle management (disposable marking)
- Export dequeue function for sound playback to retrieve incoming network audio
- Export release function to return buffers to the free pool
- Provide inline helper to check buffer disposal requirement

## External Dependencies
- `#include "cseries.h"` ΓÇö platform abstraction; provides `byte`, `uint32`, SDL integration
- Implementation of exported functions: defined elsewhere (likely `network_speaker_sdl.c`)

# Source_Files/Network/network_speaker_shared.cpp
## File Purpose
Handles receiving and processing network audio from remote players in multiplayer games. Manages audio decompression (Speex codec), team-based audio filtering, and muting of individual player microphones via UI-accessible controls.

## Core Responsibilities
- Receive incoming network audio data from remote players
- Decode Speex-compressed audio frames if enabled
- Enforce team-audio-only flag (squad-chat filtering)
- Maintain and toggle mute list for individual players
- Queue decompressed audio for playback to the local player
- Provide user feedback (screen messages) on mute/unmute actions

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö platform/endianness definitions, standard types
  - `network_sound.h` ΓÇö `queue_network_speaker_data()` declaration
  - `network_data_formats.h` ΓÇö `network_audio_header_NET`, `netcpy()` overloads
  - `network_audio_shared.h` ΓÇö `network_audio_header` struct, audio format constants, flags
  - `player.h` ΓÇö `get_player_data()`, `local_player` extern
  - `shell.h` ΓÇö `screen_printf()` for UI feedback
  - `speex/speex.h`, `network_speex.h` (conditional, SPEEX flag) ΓÇö `gDecoderState`, `gDecoderBits`, Speex codec functions
  - `<set>` ΓÇö std::set container

- **Defined elsewhere:**
  - `local_player` ΓÇö global pointer to local player data structure
  - `gDecoderState`, `gDecoderBits` ΓÇö global Speex decoder state/bitstream (from `network_speex.h`)
  - `queue_network_speaker_data()` ΓÇö enqueues PCM for playback (defined in another network speaker implementation file)
  - `netcpy()` ΓÇö byte-order conversion utilities for network structures

# Source_Files/Network/network_speex.cpp
## File Purpose
Wrapper for the Speex audio codec, managing encoder and decoder initialization for networked voice communication in Aleph One. Handles codec state setup, quality configuration, and audio preprocessing (noise reduction, automatic gain control).

## Core Responsibilities
- Initialize Speex encoder with fixed bitrate (quality 3 = 8 kbps) and complexity settings
- Initialize Speex decoder with enhancement mode enabled
- Set up audio preprocessing: denoise and AGC (automatic gain control)
- Manage global encoder/decoder state lifecycle and bit buffer allocation
- Provide cleanup routines for both encoder and decoder

## External Dependencies
- **Speex codec library:** `<speex/speex_preprocess.h>` (preprocessing), `speex_encoder_*`, `speex_decoder_*`, `speex_bits_*` (from `network_speex.h` includes `<speex/speex.h>`)
- **Network audio parameters:** `kNetworkAudioSampleRate` from `network_audio_shared.h` (8000 Hz constant)
- **Standard includes:** `cseries.h` (project common header)
- **Conditional:** `preferences.h` included but not used in this file

# Source_Files/Network/network_speex.h
## File Purpose
Header declaring Speex audio codec state management for network audio transmission in Aleph One (Marathon game engine). Provides initialization and cleanup routines for encoding/decoding game audio over the network using the Speex codec library.

## Core Responsibilities
- Declare global encoder/decoder state objects and bit buffers
- Export initialization functions for encoder and decoder setup
- Export cleanup functions for graceful shutdown
- Conditionally compile audio codec support based on SPEEX define

## External Dependencies
- `cseries.h` ΓÇö cross-platform compatibility layer (SDL, macOS emulation)
- `<speex/speex.h>` ΓÇö Speex codec library (encoder/decoder core API)
- `<speex/speex_preprocess.h>` ΓÇö Speex audio preprocessing (noise reduction)

# Source_Files/Network/network_star.h
## File Purpose
Defines the hub-and-spoke network architecture interface for Marathon: Aleph One multiplayer games. Manages both hub (server/coordinator) and spoke (client) networking roles, including packet handling, tick synchronization, action flag distribution, and lossy streaming data channels.

## Core Responsibilities
- Define message type constants for star topology protocol communication
- Declare hub-side functions: initialization, packet receiving, graceful shutdown, preferences management
- Declare spoke-side functions: initialization, packet receiving, timing queries, lossy streaming, shutdown
- Manage tick-based action flag queues for synchronized player input distribution
- Measure and report network latency for both hub-to-spoke and spoke-to-spoke (via hub) connections
- Support lossy streaming channels (e.g., voice/microphone data) with selective recipient targeting
- Provide XML-based preference serialization for both hub and spoke configurations

## External Dependencies
- **TickBasedCircularQueue.h**: Template classes `ConcreteTickBasedCircularQueue<T>`, `WritableTickBasedCircularQueue<T>` for tick-indexed action buffers
- **ActionQueues.h**: Related action queue management (referenced but not directly used in this header)
- **sdl_network.h**: Network primitives (`DDPPacketBufferPtr`, `NetAddrBlock`, UDP socket API)
- **map.h** (conditional): Defines `TICKS_PER_SECOND` (30); skipped in standalone hub builds
- **XML_ElementParser**: Preferences parser (declared; defined elsewhere)
- **Standard C**: stdio.h for file I/O in preference serialization

# Source_Files/Network/network_star_hub.cpp
## File Purpose
Implements the hub component of a star-topology network protocol for Aleph One game engine. Manages packet routing, player state, action flag distribution, and network timing synchronization across connected game clients (spokes).

## Core Responsibilities
- Hub initialization, cleanup, and periodic tick processing
- Receiving and parsing game data packets from remote players (spokes)
- Tracking player connectivity, acknowledgments, and netdead status
- Distributing action flags to all connected players with timing adjustments
- Managing lossy byte stream data buffering and routing
- Detecting player timeout and enforcing network death
- Bandwidth reduction through recovery sends and late flag handling

## External Dependencies

- **Includes:** `network_star.h`, `TickBasedCircularQueue.h`, `network_private.h`, `mytm.h`, `AStream.h`, `Logging.h`, `WindowedNthElementFinder.h`, `CircularByteBuffer.h`, `XML_ElementParser.h`, `SDL_timer.h`, `crc.h`, `player.h`
- **External symbols:** `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()`, `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `spoke_received_network_packet()`, `calculate_data_crc_ccitt()`, `local_random()`, logging macros

# Source_Files/Network/network_star_spoke.cpp
## File Purpose
Implements the spoke (client) side of the star topology network protocol for Aleph One multiplayer games. Handles game data packet exchange with the hub, action flags queuing/acknowledgement, lossy byte stream distribution, and dynamic timing synchronization.

## Core Responsibilities
- Initialize/cleanup spoke networking state and timer task
- Process incoming game data packets from hub (CRC validation, message dispatch, action flags deserialization)
- Queue and transmit local player action flags to hub
- Track and dequeue hub acknowledgements for sent flags
- Manage per-player action flag queues and net-dead state
- Measure and apply timing adjustments for clock synchronization
- Buffer and distribute lossy byte streams to specified player sets
- Maintain rolling-window latency statistics (30 ticks / ~1 second)
- Handle NAT-friendly identification packets (spokes identify themselves to hub)

## External Dependencies
- **Network layer:** `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()` (defined elsewhere).
- **Timers:** `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `take/release_mytm_mutex()` from `mytm.h`.
- **Serialization:** `AIStreamBE`, `AOStreamBE` from `AStream.h`.
- **Utilities:** `WindowedNthElementFinder<int32>`, `CircularByteBuffer`, `CircularQueue<T>` from headers.
- **Game logic callbacks:** `make_player_really_net_dead(size_t)`, `call_distribution_response_function_if_available(byte*, uint16, int16, uint8)` (declared extern).
- **Input:** `parse_keymap()` from `vbl.h`.
- **Player data:** `NetGetPlayerData(size_t)` returns `player_info*` for team/player queries.
- **Logging:** `logContextNMT()`, `logDumpNMT*()`, `logWarningNMT*()`, etc.
- **CRC:** `calculate_data_crc_ccitt(byte*, uint32)`.
- **Protocol constants:** `kPROTOCOL_TYPE` from `network_private.h`; message types from `network_star.h`.

# Source_Files/Network/network_udp.cpp
## File Purpose
Implements UDP-based datagram networking for the Aleph One game engine, replacing the original AppleTalk DDP protocol. Provides socket management, packet reception via a dedicated background thread, and packet transmission to remote machines.

## Core Responsibilities
- Initialize/shutdown the UDP networking module
- Open and close UDP sockets on specified ports
- Allocate and deallocate frame buffers for outgoing packets
- Receive incoming UDP packets in a background thread
- Invoke registered packet handlers when data arrives
- Send UDP frames to remote addresses
- Manage socket sets and thread lifecycle with proper synchronization

## External Dependencies
**SDL_net:** `SDLNet_AllocPacket`, `SDLNet_FreePacket`, `SDLNet_UDP_Open`, `SDLNet_UDP_Close`, `SDLNet_AllocSocketSet`, `SDLNet_FreeSocketSet`, `SDLNet_UDP_AddSocket`, `SDLNet_CheckSockets`, `SDLNet_UDP_Recv`, `SDLNet_UDP_Send`, `SDL_SwapBE16`

**SDL:** `SDL_Thread`, `SDL_CreateThread`, `SDL_WaitThread`

**Game engine internals:** `mytm_mutex` (mutual exclusion), `BoostThreadPriority` (thread scheduling), `fdprintf` (debug logging), `PacketHandlerProcPtr` callback type

**C standard library:** `malloc`, `free`, `memcpy`, `memset`

**Defined elsewhere:** `ddpMaxData` (max packet size), `DDPFrame`, `DDPPacketBuffer`, `kPROTOCOL_TYPE` constants

# Source_Files/Network/NetworkGameProtocol.cpp
## File Purpose
Implementation file for the NetworkGameProtocol abstract base class. Currently contains only includes and no implementation; the interface is defined in the corresponding header and serves as the protocol contract for network game communication in the Aleph One game engine.

## Core Responsibilities
- Define and export the `NetworkGameProtocol` abstract interface for network game protocol implementations
- Establish the contract for game state synchronization across the network
- Define the lifecycle for network game sessions (Enter, Exit1, Exit2)
- Specify packet handling and information distribution mechanisms
- Manage unconfirmed action flags for client-side prediction

## External Dependencies
- `network_private.h` ΓÇô internal network module definitions
- `DDPPacketBuffer`, `NetTopology` ΓÇô types defined elsewhere (DDP packet and network topology)
- Pure virtual interface; all methods implemented by derived classes

# Source_Files/Network/NetworkGameProtocol.h
## File Purpose
Abstract base class (interface) defining the contract for a generic network game protocol implementation. Decouples game networking logic from specific protocol implementations (e.g., ring protocol). Part of Aleph One's multi-player networking subsystem.

## Core Responsibilities
- **State management**: Enter/exit game network sessions with initialization and cleanup
- **Information distribution**: Send game/player data across the network with loss-model control (lossy vs. lossless)
- **Synchronization**: Establish and break network game synchronization across peers
- **Packet handling**: Process incoming network packets dispatched from the transport layer
- **Time coordination**: Provide authoritative network time for game frame synchronization
- **Action prediction**: Manage unconfirmed action flags for client-side prediction before network acknowledgment

## External Dependencies
- **Included:** `network_private.h` (provides `DDPPacketBuffer`, `NetTopology` types; also includes `cstypes.h`, `sdl_network.h`, `network.h`).
- **External symbols used:** `DDPPacketBuffer`, `NetTopology` (structs defined in network_private.h).

# Source_Files/Network/RingGameProtocol.cpp
## File Purpose
Implements the old ring-topology network game protocol for Aleph One (Marathon engine). Players arranged in a ring pass action-flag packets sequentially to a server; the server broadcasts timing and synchronization to coordinate gameplay across all clients in the network.

## Core Responsibilities
- Maintain ring topology state (upring/downring neighbors, sequence tracking)
- Queue and transmit local player action flags via ring packets with ACK/retry
- Synchronize game startup (ring must complete two cycles before game starts)
- Implement adaptive latency tuning to balance responsiveness vs. jitter tolerance
- Handle player dropout by removing dead players from the ring and reassigning server role if needed
- Process incoming ring packets and distribute action flags to player queues
- Manage three background task types: resend (ACK timeout), server, and queueing (local input capture)

## External Dependencies
- **ActionQueues.h:** `GetRealActionQueues()` for accessing local player's action queue
- **player.h:** Player constants and queue accessors
- **network.h, network_private.h:** Topology, packet headers, state constants
- **network_data_formats.h:** `netcpy()` for pack/unpack; `SIZEOF_NetPacket`, etc.
- **mytm.h:** `myTMSetup`, `myXTMSetup`, `myTMRemove` for background task scheduling
- **map.h:** `TICKS_PER_SECOND` constant
- **vbl.h:** `parse_keymap` (called from elsewhere)
- **interface.h:** `process_action_flags()`
- **XML_ElementParser.h:** XML config parsing base class
- **Logging.h:** `logNote1`, `logDump2`, `logWarning2`
- **SDL_net (via sdl_network.h):** DDP networking (frames, sockets)

# Source_Files/Network/RingGameProtocol.h
## File Purpose
Header file defining the RingGameProtocol class, which implements the NetworkGameProtocol interface for ring-topology multiplayer networking in Aleph One. Provides abstraction for synchronization, packet handling, and information distribution in network games using a ring network topology.

## Core Responsibilities
- Implement NetworkGameProtocol abstract interface for ring topology
- Manage network game initialization (Enter) and cleanup (Exit1/Exit2)
- Handle synchronization of network state and game ticks
- Route and distribute game information across the network
- Process incoming network packets
- Track and manage unconfirmed action flags for client-side prediction
- Provide static factory method for XML configuration parsing

## External Dependencies
- **Base class**: `NetworkGameProtocol` (abstract interface)
- **Type declarations** (defined elsewhere): `NetTopology`, `DDPPacketBuffer`, `XML_ElementParser`
- **Included**: `stdio.h` (C standard I/O, used by WriteRingPreferences)
- **Network internals**: Dependencies via included `NetworkGameProtocol.h` ΓåÆ `network_private.h`

# Source_Files/Network/SDL_netx.cpp
## File Purpose
Extends SDL_net with UDP broadcast capabilities for cross-platform networking. Provides functions to enable broadcast mode on sockets, send packets to all available broadcast addresses, and discover system interface broadcast addresses on non-Windows platforms.

## Core Responsibilities
- Enable/disable SO_BROADCAST socket option on UDP sockets
- Send UDP packets to broadcast addresses (platform-aware implementation)
- Collect and cache broadcast addresses from system interfaces via `ioctl()` (POSIX platforms)
- Handle platform-specific differences (Windows, BeOS, Mac, Linux, PSP, Solaris)
- Manage file-static broadcast address cache to avoid repeated system queries

## External Dependencies
- **SDL_net**: `UDPsocket`, `UDPpacket` types; `SDLNet_UDP_Send()` function
- **Platform socket APIs**:
  - Windows: `<winsock.h>` (setsockopt, socket types)
  - POSIX: `<sys/socket.h>`, `<netinet/in.h>`, `<sys/ioctl.h>`, `<net/if.h>` (ioctl, ifreq, sockaddr_in)
  - Solaris: BSD_COMP macro for SIOC* definitions

# Source_Files/Network/SDL_netx.h
## File Purpose
Header file declaring the public API for SDL_netx, a network extension library that adds broadcast capabilities to SDL_net. Provides functions to enable/disable UDP broadcast mode and send broadcast packets across all available network interfaces.

## Core Responsibilities
- Enable broadcast mode on UDP sockets
- Disable broadcast mode on UDP sockets
- Send UDP packets to all available broadcast addresses in the system
- Manage socket capabilities for broadcast transmission

## External Dependencies
- `SDL_net.h` ΓÇô provides `UDPsocket`, `UDPpacket` types and UDP socket management

# Source_Files/Network/SSLP_API.h
## File Purpose
Public API header for SSLP (Simple Service Location Protocol), a service discovery library for locating and advertising network services across non-AppleTalk networks. Built as a callback-driven alternative to NBP (Name Binding Protocol) for the Aleph One game engine, leveraging SDL_net for cross-platform IP networking.

## Core Responsibilities
- Define the `SSLP_ServiceInstance` data structure for representing discoverable network services
- Provide callback-based APIs for searching for remote services with found/lost/name-changed notifications
- Provide APIs for advertising local services to the network and hinting specific remote hosts
- Supply a pump function for non-threaded implementations requiring periodic processing time
- Define callback function pointer types for service status change events

## External Dependencies
- **SDL_net.h**: Provides `IPaddress` struct for network addresses.
- **Implementation file** (not shown): Contains state management, UDP broadcast logic, and packet format definitions.

# Source_Files/Network/SSLP_limited.cpp
## File Purpose
Implements the Simple Service Location Protocol (SSLP) for network service discovery in Aleph One. Provides single-threaded, pump-based service discovery and advertisement using SDL_net UDP broadcast, replacing AppleTalk NBP for IP networks. Built as a simplified, non-threaded alternative to support dynamic player location in network games.

## Core Responsibilities
- Service instance discovery: locate and track remote service instances via broadcast FIND/HAVE messages
- Service advertisement: allow local services to be discovered by responding to FIND queries
- Instance lifecycle management: maintain linked list of discovered services with timeout-based expiration
- Network message handling: receive and dispatch FIND, HAVE, and LOST packet types with validation
- Cross-subnet hinting: unicast HAVE announcements to specific hosts in isolated broadcast domains
- Packet serialization: pack/unpack protocol messages with endian-neutral binary format
- Callback invocation: trigger application callbacks for discovered/lost/renamed instances from main thread

## External Dependencies
- **SDL**: `SDL_GetTicks()`, `SDL_SwapBE16()`, `SDL_SwapBE32()` (endian swap), `SDL_thread.h` (included but not usedΓÇölegacy)
- **SDL_net**: `SDLNet_UDP_Open()`, `SDLNet_UDP_Close()`, `SDLNet_AllocPacket()`, `SDLNet_FreePacket()`, `SDLNet_UDP_Send()`, `SDLNet_UDP_Recv()`
- **SDL_netx** (broadcast extensions): `SDLNetx_EnableBroadcast()`, `SDLNetx_UDP_Broadcast()`
- **C stdlib**: `malloc()`, `free()`, `memcpy()`, `memset()`, `strncpy()`, `strncmp()`
- **Aleph One Logging**: `logTrace()`, `logTrace4()`, `logNote()`, `logNote1()`, `logNote2()`, `logAnomaly()`, `logContext()`, `Logger` class

# Source_Files/Network/SSLP_Protocol.h
## File Purpose
Protocol definition header for SSLP (Simple Service Location Protocol), a UDP-based service discovery mechanism for locating network game servers. Defines packet structure, constants, and operation semantics for the Aleph One game engine's network player discovery system.

## Core Responsibilities
- Define SSLP protocol constants (magic, version, message types, port)
- Specify the wire format of SSLP_Packet for cross-platform compatibility
- Document protocol operation (FIND/HAVE/LOST message flow)
- Define service type/name string length constraints
- Ensure compiler-independent struct packing for network serialization

## External Dependencies
- `<SDL_net.h>` ΓÇô SDL cross-platform IP networking library (Uint32, Uint16 types)

---

**Notes:**
- Packet uses C-string semantics for service_type/name fields; comparison/copying stops at first `\0`.
- NULL termination is *not* required if all SSLP_MAX_*_LENGTH bytes are used.
- All numeric fields in wire format are network (big-endian) byte order.
- Magic and version fields catch protocol mismatches; message type field determines packet role (FIND/HAVE/LOST).

# Source_Files/Network/StarGameProtocol.cpp
## File Purpose

Glue layer bridging the star-topology network protocol implementation to the Aleph One game engine. Manages network synchronization, packet routing, and action flag distribution for hub-and-spoke multiplayer games.

## Core Responsibilities

- Adapt legacy action queues to tick-based circular queue interface
- Manage network lifecycle (initialization, synchronization, cleanup)
- Route incoming packets to hub or spoke handlers based on network role
- Distribute game information with lossy streaming support
- Track unconfirmed action flags and network timing
- Provide XML configuration parsing for network preferences

## External Dependencies

- `network_star.h` ΓÇö Hub/spoke protocol functions (`hub_initialize`, `spoke_initialize`, `spoke_get_net_time`, etc.)
- `TickBasedCircularQueue.h` ΓÇö `WritableTickBasedCircularQueue<action_flags_t>`
- `player.h` ΓÇö `GetRealActionQueues()`
- `interface.h` ΓÇö `process_action_flags()` (used in adapter)
- Base class `NetworkGameProtocol` (interface contract)
- Defined elsewhere: `NetTopology`, `DDPPacketBuffer`, `NetDistributionInfo`, `NetAddrBlock`

# Source_Files/Network/StarGameProtocol.h
## File Purpose
Header file defining `StarGameProtocol`, a concrete implementation of the `NetworkGameProtocol` interface for star-topology network game protocol handling. Provides the contract for initialization, packet handling, state synchronization, and information distribution in a star-topology multiplayer game.

## Core Responsibilities
- Implement the `NetworkGameProtocol` virtual interface for star-topology networking
- Initialize and shut down the protocol layer (two-phase exit)
- Distribute game information/events to all players with optional filtering
- Synchronize and desynchronize game state across the network
- Handle incoming network packets and route them appropriately
- Manage network clock/timing
- Track unconfirmed action flags for client-side prediction
- Parse protocol configuration via XML

## External Dependencies
- **Includes**: `NetworkGameProtocol.h` (base class), `<stdio.h>`
- **Forward declarations**: `XML_ElementParser`
- **Symbols from other headers** (not defined here): `NetTopology`, `DDPPacketBuffer`
- **External functions**: `DefaultStarPreferences()`, `WriteStarPreferences(FILE*)` ΓÇö configure and persist star protocol preferences

# Source_Files/Network/Update.cpp
## File Purpose
Implements an online update checker for Aleph One that runs in a background thread. Connects to a remote server, retrieves the latest version information for the current platform, and reports whether an update is available by comparing versions.

## Core Responsibilities
- Manages the update-check singleton instance and its lifecycle
- Spawns and joins SDL worker threads for non-blocking network operations
- Establishes TCP connections and sends HTTP GET requests to the update server
- Parses HTTP response headers to extract version metadata
- Compares remote version (date-based string) against the local version constant
- Tracks update check status (checking, failed, available, or not available)

## External Dependencies
- **SDL_net**: `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()` for networking
- **SDL threads**: `SDL_CreateThread()`, `SDL_WaitThread()` for background execution
- **Boost**: `boost::tokenizer` and `boost::char_separator` for HTTP response parsing
- **alephversion.h**: Defines `A1_DATE_VERSION`, `A1_DISPLAY_VERSION`, `A1_UPDATE_PLATFORM` (platform-specific build macros)

# Source_Files/Network/Update.h
## File Purpose
Declares a singleton `Update` class that checks for new versions of the Aleph One game engine asynchronously. Manages update-check status and new version information, running the check in a background SDL thread to avoid blocking the main application.

## Core Responsibilities
- Singleton instance management for centralized update checking
- Status tracking (checking, failed, available, not available)
- Storage of new version metadata (date version, display version)
- Background thread management for non-blocking update checks
- Public interface for querying current status and available version

## External Dependencies
- `<SDL_thread.h>` / `<SDL/SDL_thread.h>` ΓÇô Cross-platform threading primitives
- `<string>` ΓÇô Standard C++ string storage
- `cseries.h` ΓÇô Project's common cross-platform utilities (likely includes assert macros, SDL setup)


