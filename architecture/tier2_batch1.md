# Architecture Overview

## Repository Shape

```
Source_Files/
Γö£ΓöÇΓöÇ CSeries/                 # Cross-platform type/utility abstraction layer
Γö£ΓöÇΓöÇ Expat/                   # XML tokenization (character classification tables)
Γö£ΓöÇΓöÇ Files/                   # File I/O, resource management, WAD serialization
Γö£ΓöÇΓöÇ GameWorld/               # Core simulation engine (entities, collision, physics)
Γö£ΓöÇΓöÇ Input/                   # Player input handling (mouse, PSP controller adapter)
Γö£ΓöÇΓöÇ Misc/                    # Configuration, preferences, UI framework, logging
Γö£ΓöÇΓöÇ Network/                 # Multiplayer networking (discovery, protocols, audio)
Γö£ΓöÇΓöÇ RenderMain/              # Rendering pipeline (visibility, sorting, rasterization)
Γö£ΓöÇΓöÇ RenderOther/             # (referenced in prompt; likely rendering utilities)
Γö£ΓöÇΓöÇ Sound/                   # Audio subsystem
Γö£ΓöÇΓöÇ XML/                     # XML parsing and game configuration
Γö£ΓöÇΓöÇ Scripting/ (Lua)         # Lua VM integration and game object bindings
Γö£ΓöÇΓöÇ TCPMess/                 # (referenced in prompt; likely network utilities)
ΓööΓöÇΓöÇ Third-Party/             # External libraries (Expat, LibNAT, Lua 5.1, SDL 1.2)
```

## Major Subsystems

### CSeries (Cross-Platform Compatibility Layer)
- **Purpose:** Abstract platform-specific APIs (macOS Toolbox, SDL, BeOS, Windows) behind portable type definitions, string handling, pixel operations, dialogs, timing, and utility macros. Enables single-source compilation across PSP, desktop SDL, macOS Classic/Carbon, and BeOS.
- **Key directories / files:** 
  - `cstypes.h` ΓÇö fixed-width integer types, 16.16 fixed-point arithmetic
  - `csstrings.h/cpp` ΓÇö Pascal/C string conversions, Mac RomanΓåöUTF-8 encoding
  - `cspixels.h` ΓÇö pixel type aliases (8/16/32-bit), RGB packing/extraction
  - `csdialogs.h/cpp` ΓÇö dialog control abstraction (Mac vs. SDL indexing reconciliation)
  - `csalerts.h/cpp` ΓÇö modal alert display with text wrapping and fallback stderr
  - `csmisc.h/cpp` ΓÇö tick counting, user input polling, timing
  - `byte_swapping.h/cpp` ΓÇö endianness-aware 16/32-bit field conversion
  - `mytm.h/cpp` ΓÇö timer task scheduling with drift-free periodic callbacks
- **Key responsibilities:**
  - Provide portable fixed-width integer types and fixed-point (16.16) arithmetic
  - Unify string APIs across Mac/SDL/Unix via `getcstr()`, `csprintf()`, resource-based `TS_GetString()`
  - Convert between RGB colors and 16/32-bit pixel formats with extraction macros
  - Display modal alert dialogs with multi-line word-wrapping
  - Manipulate dialog controls uniformly across Mac OS and SDL implementations
  - Perform high-resolution machine tick counting for frame timing and profiling
  - Schedule periodic timer task callbacks with mutex protection and drift correction
  - Swap byte order on multi-byte fields during binary data deserialization
  - Provide common utility macros (min/max, bit operations, power-of-two, bounds checking)
- **Key dependencies:** SDL (core, event, threading, timer), macOS Carbon/Cocoa frameworks (conditional), BeOS SupportDefs (conditional)

### Expat (XML Character Classification)
- **Purpose:** Pre-computed Unicode character classification bitmaps for the Expat XML tokenizer. Enables efficient character-by-character validation during tokenization without dynamic computation on PSP's constrained CPU.
- **Key directories / files:**
  - `Expat/xmltok/nametab.h` ΓÇö page-indexed bitmap arrays for XML name character rules
- **Key responsibilities:**
  - Encode XML specification character classification rules as compact bitmaps (nameStart vs. name continuation)
  - Provide sparse Unicode coverage via page indirection to minimize 32MB PSP RAM footprint
  - Support fast O(1) character validation lookups during tokenization
- **Key dependencies:** None (standalone data tables)

### PBProjects (Build Configuration & Platform Abstraction)
- **Purpose:** Autoconf-based compile-time feature detection (optional libraries), platform-specific path configuration, precompiled header aggregation, and macOS Cocoa GUI integration. Centralizes build-time decisions for portability across Unix/Linux, Windows, macOS, and PSP.
- **Key directories / files:**
  - `config.h` ΓÇö Autoconf-generated `HAVE_*` feature macros (OpenGL, SDL_image, SDL_net)
  - `confpaths.h` ΓÇö compile-time data directory paths (macOS .app bundle structure)
  - `precompiled_headers.h` ΓÇö aggregated SDL and framework includes for build optimization
  - `SDLMain.h` ΓÇö Objective-C Cocoa application controller with menu handlers
- **Key responsibilities:**
  - Auto-generate and maintain feature availability macros consumed by conditional compilation
  - Define platform-specific resource/data directory paths
  - Aggregate frequently-included headers into PCH for compiler optimization
  - Provide macOS GUI entry point with menu-driven lifecycle handlers (new/open/save game, preferences)
  - Abstract platform differences via preprocessor guards (`__PSP__`, platform-specific macros)
- **Key dependencies:** Autoconf build system, SDL multimedia libraries, macOS frameworks (Cocoa, Carbon), game engine module headers

### Files (File I/O & Resource Management)
- **Purpose:** Cross-platform file I/O, binary serialization with explicit endianness control, WAD file format processing, game persistence (save/load), and physics data loading for networked play.
- **Key directories / files:**
  - `FileHandler.h/cpp` ΓÇö FileSpecifier, OpenedFile, LoadedResource abstractions
  - `AStream.h/cpp` ΓÇö type-safe binary serialization with big/little-endian support
  - `Packing.h/cpp` ΓÇö byte-order conversion (StreamToValueBE/LE for 16/32-bit)
  - `tags.h` ΓÇö game data typecode identifiers (4-character WAD tags)
  - `wad.h/cpp` ΓÇö WAD file format I/O, versioning, checksums, tagged data extraction
  - `game_wad.h/cpp` ΓÇö game save/load and map persistence
  - `find_files.h/cpp` ΓÇö file searching with type-code filtering and recursive traversal
  - `resource_manager.h/cpp` ΓÇö resource fork handling (AppleSingle, MacBinary II, raw)
  - `crc.h/cpp` ΓÇö CRC-32 and CRC-CCITT checksum generation
- **Key responsibilities:**
  - Abstract file I/O primitives (FileSpecifier) for Windows, Mac, Unix, PSP
  - Provide unified resource fork handling for macOS-compatible formats on non-Mac platforms
  - Implement binary serialization with explicit big-endian file format vs. native byte order conversion
  - Parse, create, modify WAD files (tagged binary data containers) with versioning and checksums
  - Manage game state persistence (save/load games, map data, player state, world objects)
  - Provide file discovery (search by typecode, checksum, modification date; recursive traversal)
  - Calculate and verify CRC-32 checksums for file integrity and patch relationships
  - Load and synchronize physics definitions (monsters, effects, weapons, projectiles) from disk and network
  - Maintain persistent player preferences in WAD format with error recovery
- **Key dependencies:** 
  - GameWorld (save/load game state structures)
  - Network (physics data transmission)
  - Sound (audio resource loading)
  - Preferences/XML (preference definitions)
  - SDL (via SDL_RWops abstraction)

### GameWorld (Core Simulation Engine)
- **Purpose:** Orchestrate the complete game loop, manage all dynamic entities (players, monsters, items, projectiles, effects, platforms, lights, media), and coordinate physics, AI, rendering, and sound. Maintains the authoritative game state with deterministic simulation for networked play.
- **Key directories / files:**
  - `marathon2.cpp` ΓÇö main `update_world()` loop, entity lifecycle management
  - `map.cpp/h` ΓÇö world/geometry management, spatial queries, collision detection
  - `map_constructors.cpp` ΓÇö map geometry initialization and serialization
  - `world.cpp/h` ΓÇö trigonometry tables, coordinate transforms, random generation
  - `player.cpp/h` ΓÇö player lifecycle, state management, inventory, serialization
  - `physics.cpp` ΓÇö player movement, collision, gravity, jumping, camera positioning
  - `monsters.cpp/h` ΓÇö monster AI, targeting, animation, serialization
  - `pathfinding.cpp` ΓÇö flood-fill path generation over polygon adjacency
  - `projectiles.cpp/h` ΓÇö projectile lifecycle, collision, detonation with area damage
  - `effects.cpp/h` ΓÇö temporary visual/audio effects management
  - `items.cpp/h` ΓÇö item spawning, pickup mechanics, inventory tracking
  - `platforms.cpp/h` ΓÇö door/moving surface simulation with collision detection
  - `lightsource.cpp/h` ΓÇö dynamic light management with animation
  - `media.cpp/h` ΓÇö hazardous liquid management (height, flow, damage)
  - `scenery.cpp/h` ΓÇö static scenery objects with destruction
  - `weapons.cpp/h` ΓÇö weapon state machines (firing, reloading, charging)
  - `dynamic_limits.cpp/h` ΓÇö configurable runtime entity limits (XML-driven)
- **Key responsibilities:**
  - Allocate, initialize, update per-tick, and remove all dynamic objects
  - Orchestrate per-frame updates across physics, AI, animation, collision, damage, serialization
  - Perform spatial queries and collision detection on polygon-based terrain
  - Initialize and persist world geometry; manage level transitions
  - Simulate player physics, inventory, weapons, damage/respawn, oxygen, powerups
  - Implement monster AI (targeting, pathfinding, combat, animation state machines)
  - Manage environmental simulation (lighting, hazardous media, moving platforms, effects)
  - Support XML-driven configuration of items, weapons, media, platforms, and dynamic limits
  - Provide tick-based action queues for networked client prediction
  - Serialize entities for save-games and network replication
- **Key dependencies:**
  - Input (action queue processing)
  - RenderMain (camera/frustum queries)
  - Sound (weapon fire, impacts, ambient audio)
  - Network (tick-based synchronization, multiplayer action queues)
  - Lua (script hooks for entity events)
  - XML (MML configuration parsing)
  - Files (map/game data loading)

### Input (Player Control)
- **Purpose:** Unify player input across desktop SDL and PSP hardware via controller-to-mouse event adapter. Captures mouse/controller state, applies physics simulation, and integrates input into the game loop.
- **Key directories / files:**
  - `mouse.h` ΓÇö cross-platform mouse interface (init, shutdown, poll, state query)
  - `mouse_sdl.cpp` ΓÇö SDL implementation (movement deltas, sensitivity, button mapping)
  - `psp_mouse_sdl.h/cpp` ΓÇö PSP adapter class; reads PSP controller state and dispatches SDL mouse events
  - `ISp_Support.h` ΓÇö legacy InputSprocket device enumeration (unused on PSP)
- **Key responsibilities:**
  - Initialize and shut down input device handling
  - Poll input state and extract movement deltas (yaw, pitch, velocity)
  - Map hardware input (PSP controller, SDL mouse) to game action flags
  - Simulate mouse events from PSP controller input (digital button-driven or analog stick modes)
  - Manage cursor position, visibility, recentering
  - Convert mouse buttons and scroll wheel to keyboard events for menus
  - Apply velocity physics (acceleration, deceleration, max velocity caps)
- **Key dependencies:**
  - `input_preferences` (sensitivity, inversion, acceleration)
  - SDL event injection
  - PSP controller (via PSPSDK sceCtrlReadBufferPositive on PSP)

### LibNAT (UPnP/NAT Traversal)
- **Purpose:** Enable applications to discover Internet Gateway Devices (routers), retrieve public IP addresses, and manage port mappings for NAT traversal via UPnP/SSDP protocols. Abstracts socket operations across Unix and Windows.
- **Key directories / files:**
  - `libnat.h` ΓÇö public C API for IGD control, device discovery, port mapping
  - `error.h` ΓÇö error/status codes and diagnostic utilities
  - `http.h` ΓÇö HTTP client for router communication
  - `ssdp.h` ΓÇö SSDP discovery protocol for UPnP device location
  - `os.h` ΓÇö platform detection and socket operation abstraction (Unix vs. Windows)
- **Key responsibilities:**
  - Discover Internet Gateway Devices via SSDP multicast on local networks
  - Retrieve public IP addresses from discovered IGDs
  - Create and delete bidirectional port mappings via UPnP for inbound connectivity
  - Abstract socket operations (UDP/TCP send/recv, connection setup) across platforms
  - Parse HTTP responses and construct UPnP service requests
  - Provide centralized error enumeration and diagnostic utilities
- **Key dependencies:** Standard C library, platform-specific socket headers (fcntl.h, sys/socket.h, winsock.h)

### Lua (Scripting VM)
- **Purpose:** Embed Lua 5.1 virtual machine for gameplay mods and cinematics. Expose game entities to scripts via C++ template bindings and dispatch Lua callbacks in response to game events. Bridges PSP-constrained game loop with extensible Lua-based AI and event logic.
- **Key directories / files:**
  - `lua_script.h/cpp` ΓÇö Lua integration layer, event callback dispatch, game constant registration
  - `lua_templates.h` ΓÇö C++ template infrastructure (L_Class, L_Container, L_Enum wrappers)
  - `lua_map.h/cpp` ΓÇö bindings for map geometry (polygons, lines, platforms, lights, media)
  - `lua_monsters.h/cpp` ΓÇö monster/enemy bindings
  - `lua_objects.h/cpp` ΓÇö map object bindings (effects, items, scenery)
  - `lua_player.h/cpp` ΓÇö player state bindings
  - `lua_projectiles.h/cpp` ΓÇö projectile bindings
  - `lua_mnemonics.h` ΓÇö string-to-integer lookup tables for game constants
  - Lua 5.1 core headers (lua.h, lualib.h, lauxlib.h, lstate.h, lvm.h)
- **Key responsibilities:**
  - Load and execute Lua scripts; manage VM state and garbage collection
  - Wrap indexed game objects (monsters, items, projectiles, players, map elements) as Lua classes
  - Dispatch game events (damage, item pickup, monster death, switch activation) to Lua handlers
  - Register item/monster/damage/effect type mnemonics as Lua globals
  - Expose player input/control to Lua via action queues
  - Provide Lua control over camera positions, orientations, and cutscene paths
  - Allow Lua to control HUD overlays and screen effects
  - Invalidate Lua references when game objects are destroyed
  - Expose music playback and volume control to Lua
- **Key dependencies:**
  - GameWorld (monsters, items, projectiles, player, map)
  - RenderMain (camera, rendering state)
  - Sound (audio subsystems)
  - Network (action queues for networked gameplay)
  - Lua 5.1 core VM
  - Files (script loading)

### Misc (Configuration, UI, Utilities)
- **Purpose:** Centralize configuration management, input polling, game state machine, dialog/widget framework, logging, profiling, and cross-cutting utility services. Bridges core game engine with SDL/PSP platform APIs.
- **Key directories / files:**
  - `preferences.cpp/h` ΓÇö configuration management (load/save game settings from XML)
  - `vbl.cpp/h` ΓÇö input polling, action flag distribution, replay recording/playback
  - `interface.cpp/h` ΓÇö high-level game state machine (transitions, initialization)
  - `sdl_dialogs.cpp/h` ΓÇö dialog manager with modal/modeless windows, event routing
  - `sdl_widgets.cpp/h` ΓÇö widget library (~25 derived classes: text, buttons, lists, toggles, file choosers)
  - `Logging.cpp/h` ΓÇö hierarchical logging with per-domain filtering and context stack
  - `ActionQueues.cpp/h` ΓÇö circular FIFO queues for player action flags
  - `Console.cpp/h` ΓÇö in-game command parser, macro expansion, carnage reporting
  - `PlayerImage_sdl.cpp/h` ΓÇö player sprite caching and composite rendering
  - `psp_sdl_profiler*.cpp/h` ΓÇö PSP performance profiling (hierarchical function timing)
  - `CircularQueue.h`, `CircularByteBuffer.h` ΓÇö generic ring buffer templates
  - `thread_priority_sdl*.cpp` ΓÇö platform-specific thread priority boosting
- **Key responsibilities:**
  - Load, parse, validate, and persist player preferences from XML
  - Poll keyboard/gamepad input; convert to standardized action flags
  - Record and playback game sessions via film files for replay
  - Manage high-level game state transitions (intro, menu, gameplay, credits)
  - Provide SDL-based dialog system with widget library and theme rendering
  - Implement hierarchical logging with per-domain severity filtering and context tracking
  - Maintain circular FIFO action queues with zombie player filtering
  - Parse in-game commands and expand macros
  - Cache player sprites and composite rendering (legs + torso)
  - Support PSP-specific performance profiling and logging
  - Provide generic utilities (circular queues, byte buffers, random generation, math macros)
  - Manipulate thread priorities across platforms (macOS, POSIX, Win32, PSP)
- **Key dependencies:**
  - Rendering (screen drawing, text rendering, theme assets)
  - GameWorld (level metadata, player data, game state)
  - Sound/Music (background music, dialog effects)
  - Network (game lists, player lists, metaserver connectivity)
  - Input (mouse, keyboard, gamepad)
  - Files (preference persistence, replay files, theme assets)
  - XML parsing (preference definitions, theme loading)
  - Lua (post-idle hooks, end-game scripts)

### ModelView (3D Model Loading & Rendering)
- **Purpose:** Load, store, and render 3D skeletal animated models in multiple file formats (Dim3 XML, Wavefront OBJ, 3D Studio Max 3DS, QuickDraw 3D). Support animation frame interpolation and multipass shader rendering.
- **Key directories / files:**
  - `Model3D.h/cpp` ΓÇö core data structures for 3D geometry, skeletal bones, animation frames
  - `ModelRenderer.h/cpp` ΓÇö rendering interface with multipass shader support
  - `Dim3_Loader.h/cpp` ΓÇö Dim3 XML format parser
  - `WavefrontLoader.h/cpp` ΓÇö Wavefront OBJ parser with vertex deduplication
  - `StudioLoader.h/cpp` ΓÇö 3D Studio Max 3DS binary format parser
  - `QD3D_Loader.h` ΓÇö QuickDraw 3D/Quesa format loader
- **Key responsibilities:**
  - Load 3D models from five file formats into unified Model3D structures
  - Store and manage skeletal bone hierarchies with parent-child relationships
  - Organize animation frames and sequences with vertex source mapping
  - Apply skeletal deformations using bone transformations and frame interpolation
  - Compute vertex and per-polygon normals with smoothing
  - Render models via multipass OpenGL shaders with texture/lighting callbacks
  - Perform depth-sorted polygon rendering when Z-buffer unavailable
  - Maintain bounding boxes and provide debug visualization
  - Handle cross-platform file I/O via FileSpecifier abstraction
- **Key dependencies:**
  - Files (FileSpecifier, OpenedFile abstractions)
  - Rendering (OpenGL context, pixel format, texture management)
  - Vector math (VecOps.h)
  - Expat (Dim3 XML parsing)
  - Platform abstractions (cseries.h)

### Network (Multiplayer Networking)
- **Purpose:** Implement complete multiplayer infrastructure: game/player discovery (metaserver, SSLP), connection management, ring and star network protocol topologies, real-time state distribution, network audio (VoIP with optional Speex), and chat messaging.
- **Key directories / files:**
  - `network.h/cpp` ΓÇö public multiplayer API and orchestration
  - `ConnectPool.h/cpp` ΓÇö non-blocking TCP connection pooling (up to 20 concurrent)
  - `network_messages.h/cpp` ΓÇö message type definitions and serialization
  - `network_data_formats.h/cpp` ΓÇö endianness conversion layer (netcpy overloads)
  - `NetworkGameProtocol.h/cpp` ΓÇö abstract base class for protocol implementations
  - `RingGameProtocol.h/cpp` ΓÇö ring-topology (sequential action-flag passing)
  - `StarGameProtocol.h/cpp` ΓÇö star-topology (hub-and-spoke with tick-based queues)
  - `network_udp.cpp` ΓÇö UDP datagram transport (SDL_net wrapper + background thread)
  - `network_lookup_sdl.h/cpp` ΓÇö SSLP service discovery registration and lookup
  - `SSLP_API.h` ΓÇö Simple Service Location Protocol for LAN discovery
  - `metaserver_dialogs.h/cpp` ΓÇö metaserver UI abstraction
  - `network_metaserver.h/cpp` ΓÇö metaserver client (login, room management, chat)
  - `network_dialogs.h/cpp` ΓÇö cross-platform UI for gather/join flows
  - `network_sound.h` ΓÇö network audio public interface
  - `network_speaker_sdl.h/cpp` ΓÇö network audio playback with Speex decoding
  - `network_microphone_sdl_*.cpp` ΓÇö platform-specific microphone capture (DirectSound, ALSA, CoreAudio, dummy)
  - `network_speex.h/cpp` ΓÇö Speex codec initialization
  - `network_games.h/cpp` ΓÇö game mode logic (CTF, King of the Hill, etc.)
  - `network_capabilities.h/cpp` ΓÇö protocol capability versioning
  - `SDL_netx.h/cpp` ΓÇö SDL_net extension for UDP broadcast
- **Key responsibilities:**
  - Register/advertise games via metaserver (Internet) and SSLP (LAN broadcast)
  - Maintain synchronized lists of available players and games
  - Manage non-blocking TCP connection pooling with retry and DNS resolution decoupling
  - Implement ring (sequential action-flag passing with server timing) and star (hub-and-spoke with tick-indexed queues) topologies
  - Distribute game state, topology updates, and action flags with endianness-safe serialization
  - Route incoming messages (chat, topology, capabilities, game data) via message type handlers
  - Support message compression (zlib) for large payloads
  - Capture microphone input (platform-specific), encode with optional Speex, distribute to remote players
  - Decode and playback received network audio with muting controls
  - Route metaserver chat (public/private/broadcast/local) and in-game chat with filtering
  - Manage gather/join game flows, player list displays, game configuration, and progress dialogs
  - Exchange protocol version information and validate feature support (Lua, compression, Speex)
  - Implement scoring rules, rankings, and game-over conditions for nine netgame types
- **Key dependencies:**
  - GameWorld (player data, game state, action flags)
  - Preferences (network_preferences, player_preferences)
  - Files (map/scenario loading for joining games)
  - Sound (mixer singleton, platform audio APIs)
  - UI/Dialog (SDL widgets, theme system)
  - Utilities (AStream.h, Message.h, logging, CRC)

### RenderMain (Rendering Pipeline)
- **Purpose:** Orchestrate complete rendering pipeline: visibility determination via BSP ray-casting, polygon depth-sorting, object placement, and rasterization to produce on-screen image. Support dual rendering backends (software rasterizer and OpenGL) with lighting, transfer modes, animated textures, and HUD overlays.
- **Key directories / files:**
  - `render.cpp/h` ΓÇö core pipeline orchestrator; entry point for frame rendering
  - `RenderVisTree.cpp/h` ΓÇö visibility culling via BSP ray-casting
  - `RenderSortPoly.cpp/h` ΓÇö depth-sorts visible polygons into back-to-front order
  - `RenderPlaceObjs.cpp/h` ΓÇö places 3D objects into sorted rendering order
  - `RenderRasterize.cpp/h` ΓÇö transforms clipped polygons to screen space
  - `Rasterizer.h` ΓÇö abstract base class for rendering backend interface
  - `Rasterizer_OGL.h` ΓÇö OpenGL implementation
  - `Rasterizer_SW.h` ΓÇö software implementation
  - `OGL_Render.cpp/h` ΓÇö OpenGL context lifecycle and coordinate transformation
  - `OGL_Setup.cpp/h` ΓÇö OpenGL detection, configuration, asset loading
  - `OGL_Textures.cpp/h` ΓÇö OpenGL texture allocation, loading, lifecycle management
  - `OGL_Model_Def.cpp/h` ΓÇö 3D model loading and transformation
  - `OGL_Faders.cpp/h` ΓÇö OpenGL fade effect rendering
  - `scottish_textures.cpp/h` ΓÇö software texture rasterizer
  - `AnimatedTextures.cpp/h` ΓÇö animated wall texture frame sequences
  - `shapes.cpp` ΓÇö shape collection loading and RLE unpacking
  - `ImageLoader.h/cpp` ΓÇö image file loading (DDS, PNG, BMP) and format conversion
  - `low_level_textures.h` ΓÇö templated pixel-level texture rasterization
  - Collection and shape definition headers
- **Key responsibilities:**
  - Coordinate multi-stage pipeline: visibility tree ΓåÆ polygon depth-sort ΓåÆ object placement ΓåÆ coordinate transformation ΓåÆ rasterization
  - Manage camera/view state (position, orientation, FOV, projection)
  - Implement BSP ray-casting visibility culling to determine visible polygons
  - Sort polygons into back-to-front depth order with screen-space clipping windows
  - Place game objects (sprites, 3D models) into sorted rendering order
  - Transform world-space coordinates to perspective-correct screen space
  - Load, manage, and render textures across multiple formats (RGBA8, DXTC, RLE-compressed)
  - Apply lighting via precalculated shading tables and per-pixel illumination lookup
  - Render transfer modes (tinting, static effects, landscape textures, fades) with platform-specific blending
  - Manage animated wall textures with frame-sequencing and independent timing
  - Render first-person weapon/HUD layer and player crosshairs
  - Support visual effects (earthquakes, fades, infravision tinting, silhouette rendering)
  - Select between OpenGL and software rendering backends based on availability
- **Key dependencies:**
  - GameWorld (map geometry, player/monster data, lighting)
  - Preferences (graphics mode, alpha blending options)
  - Shape collections and bitmaps (indexed by shape descriptors)
  - Rendering effects (ViewControl.h for FOV, landscape effects)
  - Files/XML (texture substitutions, 3D models, fog parameters)
  - ModelView (3D skeletal model rendering)

---

## Key Runtime Flows

### Initialization

```
Game Startup (main)
  Γåô
PBProjects: Load config.h feature macros, set confpaths.h resource paths
  Γåô
SDL: Initialize SDL core (video, audio, input) via SDL_Init()
  Γåô
Preferences (Misc): Load preferences.cpp XML config ΓåÆ populate graphics/sound/input/network globals
  Γåô
Logging (Misc): Initialize hierarchical logging with context stack; open platform-specific log file
  Γåô
CSeries: Initialize platform abstractions (fixed-point constants, timer task mutex, tick counter)
  Γåô
Files: Initialize resource manager stack, open resource file contexts, lazy-init CRC lookup tables
  Γåô
Input: Call enter_mouse() to initialize mouse grabbing; initialize PSP controller adapter (psp_mouse_sdl.cpp)
  Γåô
RenderMain: Allocate render memory once; initialize OGL_Setup() if OpenGL enabled
  Γåô
GameWorld: Initialize weapon manager, physics model setup, effect definition loading from XML
  Γåô
Lua (if HAVE_LUA): Create Lua state, register game constant mnemonics, load scripts from MML
  Γåô
Sound/Audio: Initialize platform audio backend (pspaudio on PSP, ALSA on Linux, etc.)
  Γåô
Network (optional): Initialize UPnP/NAT (LibNAT) for port mapping; register SSLP discovery callbacks
  Γåô
Misc (interface.cpp): Set initial game state (intro or menu); install vbl.cpp timer task (~30 Hz)
  Γåô
Main game loop ready
```

### Per-Frame / Main Loop

```
Each Frame (~30 Hz on PSP, 60 Hz target on desktop):
  Γåô
Misc (vbl.cpp): Timer task fires: poll keyboard/gamepad state ΓåÆ apply physics (acceleration, clamping) ΓåÆ 
                 latch double-clicks ΓåÆ enqueue action flags into ActionQueues (per player)
  Γåô
Misc (interface.cpp): Process game state transition if pending (e.g., intro ΓåÆ menu, menu ΓåÆ gameplay)
  Γåô
Input (recorded via replay if playback active): Dequeue action flags from ActionQueues; apply to player
  Γåô
GameWorld (marathon2.cpp): update_world() called with action flags
    Γö£ΓöÇ Player update: Apply input ΓåÆ update physics (movement, collision, jumping) ΓåÆ weapon state ΓåÆ camera
    Γö£ΓöÇ Monster updates: Update AI (targeting, pathfinding, behavior state machine) ΓåÆ movement ΓåÆ animation
    Γö£ΓöÇ Projectile updates: Advance position ΓåÆ collision detection ΓåÆ detonation with area damage
    Γö£ΓöÇ Effect updates: Animation frame advance ΓåÆ lifetime countdown ΓåÆ removal
    Γö£ΓöÇ Platform updates: Movement with obstruction detection ΓåÆ player/monster interaction
    Γö£ΓöÇ Lua event dispatch (if HAVE_LUA): Call L_Call_Frame hooks ΓåÆ check entity lifecycle events
    Γö£ΓöÇ Polygon triggers: Check damage zones, monster activation, objective conditions
    Γö£ΓöÇ Serialization: Pack predicted game state into network buffers (if multiplayer)
    ΓööΓöÇ Sound callbacks: Queue ambient/environmental audio playback
  Γåô
Network (if multiplayer): 
    Γö£ΓöÇ Protocol pump: Receive and process incoming packets (ring or star topology)
    Γö£ΓöÇ Distribute action flags to game logic via process_action_flags()
    Γö£ΓöÇ Send/receive game state updates, topology changes, chat messages
    Γö£ΓöÇ Network audio: Batch-send microphone samples if captured; pull decoded audio from speaker buffer
    ΓööΓöÇ Metaserver: Exchange chat messages, player list updates (gathering only)
  Γåô
RenderMain (render.cpp): render_view() called with current view_data
    Γö£ΓöÇ RenderVisTree: BSP ray-cast visibility culling ΓåÆ determine visible polygons
    Γö£ΓöÇ RenderSortPoly: Depth-sort visible polygons into back-to-front order with clipping windows
    Γö£ΓöÇ RenderPlaceObjs: Project 3D objects (sprites, models) to screen space; integrate into sort order
    Γö£ΓöÇ RenderRasterize: Transform clipped polygons to screen space; delegate to rasterizer backend
    ΓööΓöÇ Rasterizer (OGL or SW): Submit textured geometry to backend; apply lighting, transfer modes, animations
  Γåô
RenderMain: Render first-person weapon/HUD layer and player crosshairs
  Γåô
RenderMain: Apply foreground effects (fade overlays, text labels)
  Γåô
Backend (OpenGL or Software): Swap buffers / flush SDL surface to display
  Γåô
Sound: Mix queued audio samples into hardware buffer; output to PSP audio or platform speaker
  Γåô
Misc (sdl_dialogs.cpp): Process SDL events (keyboard, mouse, window) ΓåÆ dispatch to active dialog/widgets
  Γåô
Frame complete; repeat
```

### Shutdown

```
Game Exit:
  Γåô
Misc (interface.cpp): Transition game state to epilogue/shutdown
  Γåô
Network: If multiplayer active, NetExit1/NetExit2 phases
    Γö£ΓöÇ Stop gathering/joining
    Γö£ΓöÇ Drop all player connections
    Γö£ΓöÇ Unregister SSLP services, disconnect from metaserver
    Γö£ΓöÇ Microphone/speaker shutdown: Release audio resources, cleanup Speex codec
    ΓööΓöÇ UDP socket cleanup, topology cleanup
  Γåô
GameWorld: Clear all dynamic entities (monsters, items, projectiles, effects)
  Γåô
Lua (if HAVE_LUA): Cleanup script hooks, destroy Lua state via lua_close()
  Γåô
Files: Close open WAD files, free CRC lookup table, close resource file contexts
  Γåô
RenderMain: Unload texture collections, free OpenGL texture/model resources (if OGL active)
  Γåô
Sound: Shutdown audio backend (pspaudio on PSP, etc.)
  Γåô
Input: Call exit_mouse() to release mouse grab, restore cursor visibility
  Γåô
Misc (preferences.cpp): Persist modified preferences to XML file via write_preferences()
  Γåô
Misc (vbl.cpp): Uninstall timer task from CSeries (mytm_sdl.cpp)
  Γåô
Misc (sdl_dialogs.cpp): Release theme resources, SDL surfaces
  Γåô
Logging: Close log file handle; dump final context stack if active
  Γåô
CSeries: Cleanup timer task mutex and any global state
  Γåô
SDL: SDL_Quit() to teardown SDL core
  Γåô
Process exit
```

---

## Data & Control Boundaries

### Global State & Ownership

| Subsystem | Data Owned | Lifetime | Access Pattern |
|-----------|-----------|----------|-----------------|
| **GameWorld** | Map geometry (polygons, endpoints, lines, sides); dynamic entities (players, monsters, items, projectiles, effects, platforms, lights, media, scenery, devices) | Per-level; entity slot-based recycling | Global arrays: `map_polygons[]`, `ObjectList`, `MonsterList`, etc.; accessor functions |
| **RenderMain** | Cached textures (mipmap chains, format conversions); 3D model meshes and bone hierarchies; animated texture frame indices; render memory buffers (visibility tree, sort lists) | Per-texture: 10ΓÇô20s age with LRU eviction; per-frame: visibility/sort trees | Lazy-load on render; purge by type age on texture pressure; per-frame allocation in RenderVisTree/RenderSortPoly |
| **Preferences (Misc)** | Player preferences (graphics mode, sensitivity, network config, audio volumes); game preferences (difficulty, netgame type) | Session lifetime; persisted to XML on exit | Extern global pointers: `graphics_preferences`, `sound_preferences`, `input_preferences`, `network_preferences`, `player_preferences`, `environment_preferences` |
| **ActionQueues (Misc)** | Per-player circular FIFO queues of action flags (movement, weapon triggers, etc.); zombie player filtering flags | Per-frame; queues reset on level transitions | Singleton via `GetRealActionQueues()`; enqueue via input subsystem; dequeue via GameWorld |
| **Network** | TCP connection pool (up to 20 concurrent) for metaserver/game servers; UDP socket; game topology (ring or star) with player slots; action queue replicas for remote players; netgame state (scores, rankings) | Session lifetime (multiplayer); dropped on exit | Per-protocol-type API; player join/leave transitions |
| **Lua (if HAVE_LUA)** | Lua VM state; indexed entity caches (monsters, items, projectiles) via L_Class template; script globals | Session lifetime; cleared on level transition | Lua stack manipulation via lua_script.cpp; C callbacks from game events; Lua calls into C via template bindings |
| **Files** | Open WAD file handles (stack of resource contexts); in-memory WAD data (map, physics, preferences); file search caches | Per-WAD (game session); preferences WAD (lifetime) | SDL_RWops abstraction; FileHandler API; lazy-load physics definitions on demand |
| **Sound/Audio** | Mixer singleton state; audio sample buffers; music playback state; network audio speaker buffer | Session lifetime | Backend-specific (pspaudio on PSP, SDL_mixer elsewhere); pull-based consumption from render loop |
| **Logging (Misc)** | Log file handle; context stack (per-domain indentation); per-domain severity thresholds | Session lifetime; context stack cleared on level exit | RAII LogContext objects; macros push/pop context; variadic `logError()`, `logWarning*()` macros |

### Control Flow Boundaries

| Boundary | Flow Direction | Synchronization |
|----------|---|---|
| **Input ΓåÆ GameWorld** | Input.test_mouse() enqueues action flags; GameWorld.update_world() dequeues and applies | Per-frame; ActionQueues circular FIFO with no explicit locks (single consumer/producer) |
| **GameWorld ΓåÆ RenderMain** | GameWorld state (entities, geometry, lighting) queried by RenderMain.render_view(); camera/frustum info returned to GameWorld for culling | Unidirectional read; no feedback control |
| **GameWorld Γåö Network** | Action flags transmitted/received; game state snapshots serialized; player topology changes broadcast | Tick-indexed action queues; network protocol pump runs between game ticks |
| **GameWorld ΓåÆ Sound** | Monster/weapon/ambient audio events queued during update_world(); mixer pulls samples on output frame | Event-based with pull-driven mixing; no explicit locks (Mixer singleton with atomics for PSP safety) |
| **RenderMain ΓåÆ TextureCache** | Textures lazy-loaded on first render; LRU eviction on memory pressure | Per-texture age tracking; eviction happens before render phase |
| **Network Γåö Metaserver** | Non-blocking TCP pool; metaserver chat routed via message handlers; game announcements asynchronous | TCP connection non-blocking; chat via message queue |
| **Lua Γåö GameWorld** | Lua hooks called synchronously during update_world(); Lua can queue player actions, modify entity state | Synchronous in event dispatch; action queue writes asynchronous for next tick |
| **Preferences Γåö UI/Dialogs** | Binder<T> abstraction two-way-syncs widget state with preference globals | Per-frame widget event dispatch; batch commit via BinderSet on dialog OK |

### Resource Lifetimes

| Resource | Allocation | Deallocation | Constraints |
|----------|-----------|--------------|-------------|
| **Map geometry** | Level load; unpacked from WAD via `map_constructors.cpp` | Level exit; cleanup via `delete_map()` | Static at runtime; no reallocation during gameplay |
| **Dynamic entities** | Slot-based arrays; initial alloc during level load; resizable via XML `dynamic_limits.cpp` | On destruction (monsters/projectiles) or level exit; slots marked free for recycling | Bounded by XML config; PSP: typical max 100 monsters, 256 projectiles, 256 effects |
| **Textures** | Lazy-load on first render via shape descriptors; mipmap chain generated | LRU eviction on memory pressure (10ΓÇô20s age thresholds per type) | OpenGL VRAM limited by PSP (4MB typical); texture atlasing required for large maps |
| **Cached models (ModelView)** | Load via Model3D::Load_*() from files; skeletal deformation computed per-frame | On model unload; reference count drops to zero | Bone reordering one-time on load for cache efficiency |
| **Network connections** | TCP connect pool on-demand; UDP socket at NetEnter(); SSLP listener thread | NetExit phases; graceful close; SSLP thread terminated | Up to 20 concurrent TCP; one UDP socket; one SSLP thread |
| **Action queues** | Allocated per player at NetEnter(); circular buffers pre-sized | NetExit phases; per-level reset clears queue pointers | Fixed size per-player (~100 flags); wrap-around on overflow |
| **Lua VM** | Created at engine startup (if HAVE_LUA); one global instance | Engine shutdown or script reload; lua_close() | Single VM instance shared across all scripts; no per-level cleanup (scripts survive level transitions) |
| **Log file** | Lazy-open on first log call; platform-specific path | Engine shutdown; explicit close() call | Single file handle; rotation handled by platform logging macros |

---

## Notable Risks / Hotspots

### Performance Bottlenecks

1. **Real-Time Rendering on PSP**: 
   - 480├ù272 LCD @ 30 Hz target requires aggressive polygon culling (RenderVisTree BSP ray-casting must complete within frame budget)
   - Software rasterizer (`scottish_textures.cpp`) on PSP without GPU support: texture-fills and lighting calculation are CPU-bound
   - **Mitigation**: Use libGU (PSP GPU) rendering path if available; fixed-point math trades precision for speed; mipmap chains reduce fill rate

2. **Network Synchronization in Multiplayer**:
   - Ring topology (sequential action-flag passing) serializes game loop across all players; latency compounds with player count
   - Star topology alleviates but requires stable hub; metadata exchange (player topology changes) can stall frame loop
   - **Mitigation**: Non-blocking ConnectPool; message compression (zlib) for large payloads; capability negotiation upfront to disable unsupported features (Lua, Speex)

3. **Memory Constraints on PSP (32MB, 64MB slim)**:
   - Texture atlasing critical; individual mipmap chains can exhaust VRAM quickly
   - MonsterList, ObjectList, ProjectileList allocations scale with XML dynamic_limits; default counts may exceed available RAM
   - Lua VM adds ~1ΓÇô2MB footprint; script memory can fragment if not careful
   - **Mitigation**: Configurable entity limits (dynamic_limits.cpp); lazy texture loading with LRU eviction; disable Lua on memory-constrained builds (`#ifdef HAVE_LUA`)

4. **Lua Script Performance**:
   - Per-frame Lua callbacks (L_Call_Frame) with large entity iteration (all monsters in view) can stall frame loop
   - Template binding overhead (L_Class metamethods) not negligible for frequent entity queries
   - **Mitigation**: Memoization of entity validity predicates; async Lua interpreter on separate thread (not implemented); selective callback registration to disable unused hooks

5. **Physics Determinism & Fixed-Point Precision**:
   - 16.16 fixed-point arithmetic limits sub-pixel precision; movement rounding can accumulate error over long distances
   - Cross-platform endianness must be consistent for replay/network sync; byte-swapping in Packing layer adds overhead
   - **Mitigation**: Validate fixed-point constants across all builds; automated determinism tests with saved game playback

### Architectural Risks

1. **Global Action Queues & Single-Threaded Game Loop**:
   - No mutex protection on ActionQueues (assumes single consumer/producer per player); race conditions possible with input polling and action dequeue in separate threads
   - Input polling (vbl.cpp timer task) and game loop run in different threads on some platforms; action queue is implicit sync point
   - **Mitigation**: Audit ActionQueues for thread safety; consider atomic operations for queue pointers if platforms use true multithreading

2. **Network Protocol Topology Lock-In**:
   - Ring vs. Star topology selected at game start; cannot switch mid-game without reconnection
   - Protocol capability negotiation (network_capabilities.h) happens pre-game; feature mismatch mid-session forces disconnect
   - **Mitigation**: Fallback protocol paths (e.g., if star hub drops, revert to ring); pre-flight capability checks before joining

3. **Lua Reference Invalidation**:
   - When GameWorld destroys monsters/projectiles, Lua_Monster::Clear() / Lua_Projectile::Clear() must be called to prevent dangling pointers
   - Missing invalidation calls lead to Lua stack corruption and crashes (hard to debug)
   - **Mitigation**: Audit all destruction paths in GameWorld for Lua invalidation calls; consider wrapper RAII destructors

4. **Preferences Persistence & XML Parsing**:
   - Preferences corrupted if app crashes during write_preferences(); no atomic file swap or rollback
   - XML parsing failures silently reset to defaults (no diagnostic logging of parse errors)
   - **Mitigation**: Atomic file operations with temp file + rename; detailed XML parse error logging (already in Expat module)

5. **Platform-Specific Code Fragmentation**:
   - PSP-specific code gated behind `#ifdef __PSP__` in multiple subsystems (Input, Audio, Network, Logging)
   - Risk of bitrot if PSP build not continuously validated (easy to break with changes that don't test on PSP hardware)
   - **Mitigation**: Separate PSP build in CI/CD pipeline; dedicated maintainer for PSP platform layer

6. **Rendering Backend Dual-Path Complexity**:
   - Both OpenGL and software rasterizer implementations must be kept in sync (OGL_Render.h vs. scottish_textures.h)
   - Bug fixes or features in one backend may not propagate to the other; visual inconsistencies likely
   - **Mitigation**: Shared rasterizer interface tests; regular testing on both codepaths; feature parity checklist

7. **CSeries & Fixed-Point Arithmetic**:
   - Fixed-point (16.16) format hardcoded throughout codebase; no wrapper type prevents accidental integer mixing
   - Bit-shift operations for fixed-point arithmetic error-prone (FIXED_ONE = 0x10000 = 65536; easy to use 0x1000 by mistake)
   - **Mitigation**: Typedef `_fixed` to wrapper struct with operator overloads (C++); lint rules to flag bare shifts of FIXED_ONE

### Configuration & Deployment Risks

1. **Autoconf Feature Detection (config.h)**:
   - Build-time feature macros (HAVE_OPENGL, HAVE_SDL_IMAGE) frozen at compile time; no runtime fallback
   - Mismatched binaries (e.g., desktop OpenGL build run on headless server) silently fail with uninformative errors
   - **Mitigation**: Runtime feature detection for optional subsystems (Network, Lua); graceful degradation on feature unavailability

2. **WAD File Format Versioning**:
   - WAD checksums validate data integrity but not format version compatibility
   - Old save games from prior engine versions may fail to deserialize if data structures change
   - **Mitigation**: Versioned WAD headers with migration functions; robust error handling in wad.cpp for format mismatches

3. **Physics Definition Synchronization**:
   - Multiplayer games load physics from disk; if server and client have different versions, netgame will desync
   - physics.cpp monsters/weapons definitions must match across all players (no pre-game validation)
   - **Mitigation**: Network::CheckPhysicsConsistency() (not evident in provided overviews); physics checksums exchanged during join handshake

### Security Concerns (if networked on untrusted networks)

1. **No encryption**: Network protocol messages sent in plaintext; metaserver chat, player topology, game state visible to network sniffers
2. **Metaserver authentication**: Simple username/password; no protection against man-in-the-middle (metaserver_messages.cpp)
3. **Script injection via Lua**: If game downloads scripts from untrusted sources, no sandbox; arbitrary code execution possible

---

This concludes the unified Architecture Overview synthesizing all twelve subsystem analyses into a coherent picture of the alephone-psp PSP port's design, runtime behavior, and known risks.
