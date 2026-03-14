# Subsystem Overview

## Purpose

This subsystem implements multiplayer networking for the Aleph One game engine, enabling players to discover, join, and play networked games together. It encompasses player/game discovery (via metaserver for Internet games and SSLP for LAN), connection management, game protocol synchronization (ring and star topologies), real-time state distribution, network audio (VoIP) with optional Speex compression, and in-game chat messaging.

## Key Files

| File | Role |
|------|------|
| network.h / network.cpp | Core public API and multiplayer game orchestration (player lifecycle, joining, topology) |
| network_private.h | Internal packet structures, topology definitions, error codes, service discovery classes |
| ConnectPool.h / ConnectPool.cpp | Non-blocking TCP connection pooling (up to 20 concurrent async connections per SDL thread) |
| network_messages.h / network_messages.cpp | Message type definitions and serialization/deserialization (inflate/deflate) via AIStream/AOStream |
| network_data_formats.h / network_data_formats.cpp | Endianness conversion layer (netcpy overloads) between native and wire-format packet structures |
| NetworkGameProtocol.h / NetworkGameProtocol.cpp | Abstract base class for protocol implementations (ring vs. star) |
| RingGameProtocol.h / RingGameProtocol.cpp | Ring-topology protocol: sequential action-flag passing with server timing sync |
| StarGameProtocol.h / StarGameProtocol.cpp | Star-topology protocol: hub-and-spoke with tick-based action queues |
| network_star.h / network_star_hub.cpp / network_star_spoke.cpp | Hub and spoke implementation for star topology |
| network_udp.cpp | UDP datagram transport (SDL_net wrapper with background thread packet reception) |
| network_lookup_sdl.h / network_lookup_sdl.cpp | SSLP service discovery registration and lookup |
| SSLP_API.h / SSLP_limited.cpp / SSLP_Protocol.h | Simple Service Location Protocol for LAN game discovery (broadcast-based) |
| metaserver_dialogs.h / metaserver_dialogs.cpp | Metaserver UI abstraction for game/player lists and chat |
| network_metaserver.h / network_metaserver.cpp | Metaserver client (login, room management, game announcement, chat routing) |
| metaserver_messages.h / metaserver_messages.cpp | Metaserver protocol messages and serialization |
| network_dialogs.h / network_dialogs.cpp | Cross-platform (SDL) UI for gathering, joining, and game setup |
| network_dialog_widgets_sdl.h / network_dialog_widgets_sdl.cpp | Custom SDL widgets for player/game lists and postgame carnage reports |
| network_sound.h | Public interface for network audio (speaker/microphone) |
| network_speaker_sdl.h / network_speaker_sdl.cpp / network_speaker_shared.cpp | Network audio playback, buffering, and Speex decoding |
| network_microphone_sdl_*.cpp | Platform-specific microphone capture (Win32 DirectSound, Linux ALSA, macOS CoreAudio, dummy fallback) |
| network_microphone_shared.h / network_microphone_shared.cpp | Shared microphone interface, format conversion, sample-rate interpolation, Speex encoding |
| network_audio_shared.h | Shared audio packet header format and constants (8000 Hz, 16-bit, mono) |
| network_speex.h / network_speex.cpp | Speex codec initialization/cleanup for audio compression |
| network_capabilities.h / network_capabilities.cpp | Protocol capability versioning (Gameworld, Star, Ring, Lua, Speex, Gatherable, ZippedData) |
| network_games.h / network_games.cpp | Game mode logic (CTF, King of the Hill, tag, rugby, etc.); scoring, rankings, game-over conditions |
| network_distribution_types.h | Network audio distribution type IDs (enumeration constants) |
| Update.h / Update.cpp | Singleton background thread for checking remote version availability |
| network_dummy.cpp | Stub implementations (disables networking if not linked) |
| SDL_netx.h / SDL_netx.cpp | SDL_net extension for UDP broadcast capabilities |

## Core Responsibilities

- **Player and game discovery**: Register/advertise games via metaserver (Internet) and SSLP (LAN broadcast); maintain synchronized lists of available players and games
- **Connection management**: Non-blocking TCP connection pooling with retry and DNS resolution decoupling for metaserver and game server connections
- **Network protocol coordination**: Implement ring (sequential action-flag passing) and star (hub-and-spoke) topologies; switch between protocols based on game configuration
- **State synchronization**: Distribute game state, player topology updates, and action flags across the network with endianness-safe serialization; handle player join/leave transitions
- **Message routing**: Dispatch incoming network messages (chat, topology, capabilities, game data) via message type handlers; support message compression (zlib) for large payloads
- **Network audio**: Capture microphone input (platform-specific), encode with optional Speex compression, and distribute to remote players; decode and playback received audio with muting controls
- **Chat system**: Route metaserver chat (public/private/broadcast/local) and in-game chat with player filtering and mute lists
- **Dialog UI**: Manage gather/join game flows, player list displays, game configuration, and progress dialogs for setup phases
- **Capability negotiation**: Exchange protocol version information between peers to validate feature support (Lua, compression, Speex, etc.)
- **Game mode management**: Implement scoring rules, rankings, and game-over conditions for nine netgame types (CTF, King of the Hill, etc.)

## Key Interfaces & Data Flow

**What this subsystem exposes:**
- **network.h**: Public multiplayer API ΓÇö `NetEnter()`, `NetGather()`, `NetGameJoin()`, `NetCheckForNewJoiner()`, `NetUpdateJoinState()`, topology/player queries, action flag distribution
- **network_dialogs.h**: Cross-platform dialog factory for UI (gather, join, setup dialogs)
- **network_sound.h**: Speaker initialization, microphone state control, idle processing hooks
- **network_games.h**: Game mode initialization, per-frame updates, ranking calculations, game-over checks
- **network_private.h**: Internal message types, packet structures, topology definitions (for subsystem-internal use)

**What this subsystem consumes:**
- **Game engine**: Player data, game state, map/scenario metadata (Scenario::instance()), game options/parameters, action flag inputs (from input subsystem)
- **UI/Dialog framework**: SDL widgets, theme/color system, progress dialogs, font/text rendering
- **Audio subsystem**: Mixer singleton for playback, platform audio APIs (DirectSound, ALSA, CoreAudio)
- **Preferences system**: network_preferences, player_preferences, environment_preferences (global persistent settings)
- **Utilities**: Stream I/O (AStream.h), message framework (Message.h), XML parsing, logging, CRC validation

## Runtime Role

**Initialization (frame 0 or UI flow):**
- `NetEnter()` called to enable networking; initializes UDP transport, topology structures, and protocol stacks (ring or star based on preferences)
- If gathering: `NetGather()` registers local service via SSLP, opens metaserver connection, accepts joining players
- If joining: `NetGameJoin()` connects to gathering server, receives map/game data, waits for server to start game
- Microphone capture starts if network audio enabled

**Per-frame (running multiplayer game):**
- Protocol pump: `RingGameProtocol::Handle()` / `StarGameProtocol::Handle()` receives and processes incoming packets
- Action flag queuing: Local player action flags captured and queued for transmission to other players
- Message dispatch: Incoming network messages (chat, topology updates, game state) routed to handlers
- Synchronization: Action flags dequeued and distributed to game logic via `process_action_flags()`
- Audio: Microphone idle processor called to batch-send audio packets; speaker pulls buffered audio for mixer playback
- Chat: Messages received from metaserver or in-game chat routed to UI display

**Shutdown:**
- `NetExit1()` / `NetExit2()` phases: Stop gathering/joining, drop players, unregister services, close connections
- Microphone/speaker shutdown: Release audio resources, cleanup Speex codec state
- Metaserver disconnect: Notify server of game closure if applicable
- UDP socket cleanup, SSLP thread termination, topology cleanup

## Notable Implementation Details

- **Endianness abstraction**: `network_data_formats.cpp` provides `netcpy()` overloads to convert between platform-native and big-endian wire format, ensuring consistency across Windows, Mac, Linux, and PSP (which uses little-endian MIPS)
- **Non-blocking connection pooling**: ConnectPool decouples DNS resolution and connection attempts into separate SDL threads to prevent blocking the main game loop during metaserver or server connection; up to 20 concurrent attempts supported
- **Message compression**: Large payloads (maps, physics, Lua scripts) compressed with zlib before transmission; uncompressed on reception in `UninflatedMessage` base class
- **SSLP broadcast-based discovery**: Replaces AppleTalk NBP (Mac-only) with cross-platform UDP broadcast for LAN game discovery; instances discovered via callback invocation, with timeout-based expiration
- **Dual protocol support**: Ring protocol (legacy, sequential action-flag passing with server timing) and Star protocol (hub-and-spoke with tick-indexed action queues) coexist; selection via game preferences and XML config
- **Platform-specific audio**: Microphone capture implemented separately for Windows (DirectSound), Linux (ALSA), macOS (CoreAudio), and PSP (via pspaudio); speaker always uses Mixer singleton
- **Speex codec**: Optional audio compression (quality 3 = 8 kbps); enabled at compile-time; includes preprocessing (denoise, automatic gain control) for voice quality
- **Game mode diversity**: Nine netgame types (Capture the Flag, King of the Hill, Tag, Rugby, etc.) each with distinct scoring rules, beacons, and game-over conditions implemented in `network_games.cpp`
- **Capability versioning**: Pre-game negotiation of feature support (Gameworld PRNG version, protocol version, Lua scripting, Speex codec, zipped data) ensures feature mismatch detection before sync
