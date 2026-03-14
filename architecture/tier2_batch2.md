# Architecture Overview

## Repository Shape

The codebase mirrors mainline Aleph One's modular organization:

- **Source_Files/** - Main source tree organized by subsystem
  - **RenderOther/** - HUD, UI, maps, camera, font rendering, screen management
  - **Sound/** - Audio mixing, playback, spatial audio, format decoders
  - **TCPMess/** - Network message I/O and dispatch
  - **XML/** - Configuration parsing via Expat
  - **CSeries/, GameWorld/, RenderMain/, Files/, Misc/, Network/, Input/** - Core game engine (not detailed in this batch)

Cross-compilation via PSPSDK with PSP-specific code gated behind preprocessor guards (`__PSP__`, `_PSP_FW_VERSION=550`). Single Makefile-based build (configure.PSP) targeting psp-gcc with `-G0` small-data model.

---

## Major Subsystems

### RenderOther

**Purpose:**
Rendering and user interface subsystem managing on-screen display of the HUD, menus, effects, text, maps, and camera systems. Bridges game world state with platform-specific rendering backends (OpenGL, SDL software rasterization) and provides cross-platform abstractions for fonts, images, and drawable primitives.

**Key directories / files:**
- Chase camera: `ChaseCam.cpp/h`
- Computer interface: `computer_interface.cpp/h`
- Screen effects: `fades.cpp/h`
- Font system: `FontHandler.cpp/h`, `sdl_fonts.cpp/h`
- HUD management: `game_window.cpp/h`, `HUDRenderer.cpp/h`, `HUDRenderer_OGL.cpp/h`, `HUDRenderer_SW.cpp/h`
- Image resources: `images.cpp/h`
- Motion sensor: `motion_sensor.cpp/h`
- Overhead maps: `overhead_map.cpp/h`, `OverheadMap_OGL.cpp/h`, `OverheadMap_SDL.cpp/h`, `OverheadMapRenderer.cpp/h`
- Screen rendering: `screen.h`, `screen_definitions.h`, `screen_drawing.cpp/h`, `screen_sdl.cpp`, `screen_shared.h`
- Utilities: `OGL_Blitter.cpp/h`, `OGL_LoadScreen.cpp/h`, `TextLayoutHelper.cpp/h`, `TextStrings.cpp/h`, `ViewControl.cpp/h`

**Key responsibilities:**
- Chase camera with spring physics and collision detection against geometry
- In-game terminal display with text rendering and input handling
- Screen fades, color tints, and gamma correction
- Font loading (bitmap and TrueType) and text rendering with style codes
- HUD buffer allocation and dirty-flag-based update optimization
- Platform-specific HUD rendering (OpenGL-accelerated or SDL software)
- PICT and CLUT image resource loading with RLE decompression
- Circular radar display tracking nearby entities
- Overhead map rendering with polygon/line geometry, entity tracking, and viewport culling
- SDL display initialization, mode switching, and buffer composition
- 2D shape, text, and polygon rendering with clipping
- Field-of-view and view effect management
- Non-overlapping rectangular text layout reservation
- String repository (replacing macOS STR# resources)

**Key dependencies (other subsystems):**
- **Map/World** (map.h): Polygon/line/endpoint geometry for rendering and collision
- **Player** (player.h): Position, orientation, equipment, health state
- **Monsters/Entities** (monsters.h): Entity positions and visibility
- **Configuration** (preferences.h): Chase camera, fade, font, HUD layout settings
- **Game State** (dynamic_world, static_world globals): Tick count, environment flags, automap arrays
- **Network** (network.h): Multiplayer restrictions, latency, compass state
- **Sound** (SoundManager.h): Audio playback for terminal and HUD interactions
- **Scripting** (lua_script.h): Lua callbacks for terminal control
- **XML**: Configuration parsing for fades, fonts, interface layout

---

### Sound

**Purpose:**
Audio playback and mixing for the Aleph One game engine, handling multi-channel real-time audio mixing, spatial audio calculations, and support for multiple compressed and uncompressed audio formats. Coordinates between SDL's audio I/O, game world state (listener position, obstruction), and built-in and external audio resources.

**Key directories / files:**
- Core: `SoundManager.cpp/h`, `Mixer.cpp/h`, `Music.cpp/h`
- Resource parsing: `SoundFile.cpp/h`
- Decoders: `Decoder.cpp/h`, `BasicIFFDecoder.cpp/h`, `MADDecoder.cpp/h`, `VorbisDecoder.cpp/h`, `SndfileDecoder.cpp/h`
- External resources: `ReplacementSounds.cpp/h`
- Definitions: `SoundManagerEnums.h`, `sound_definitions.h`, `song_definitions.h`

**Key responsibilities:**
- Load and decode audio in WAV, AIFF, MP3 (libmad), Ogg/Vorbis, and FLAC (libsndfile) formats via format-agnostic Decoder factory
- Initialize and manage SDL audio subsystem with configurable sample rate, bit depth, and channel count
- Real-time mixing of concurrent audio channels with linear interpolation for pitch shifting
- Format conversion during mixing (8/16-bit, mono/stereo, signed/unsigned, endianness adaptation)
- 3D spatial audio: stereo panning, distance-based volume attenuation, listener obstruction detection
- Intelligent channel allocation with priority queues and conflict detection
- Music playback independent of effects with fade-out support and level-specific playlists
- External sound file replacement via hash-table lookup
- Network microphone audio dequeue and playback with dynamic buffer lifecycle
- System 7 big-endian sound header parsing (22-byte standard and 64-byte extended formats)
- Ambient and random sound source management with active culling

**Key dependencies (other subsystems):**
- **SDL audio**: Audio device context and callback interface
- **SDL_net**: Network audio component
- **Game world**: Listener position/orientation, obstruction queries, ambient source enumeration
- **Sound resource files**: System 7 `.snd2` format definitions
- **XML**: Sound remapping and customization configuration
- **Network**: Microphone audio buffer dequeue

---

### TCPMess

**Purpose:**
Implements TCP-based bidirectional message communication for networked gameplay. Manages non-blocking socket I/O, message serialization/deserialization, type-based routing, and connection lifecycle across multiple endpoints.

**Key directories / files:**
- Socket management: `CommunicationsChannel.h/cpp`
- Message protocol: `Message.h/cpp`, `MessageInflater.h/cpp`
- Message routing: `MessageDispatcher.h/cpp`, `MessageHandler.h/cpp`

**Key responsibilities:**
- Establish and manage TCP socket connections with non-blocking I/O
- Buffer incoming/outgoing messages with header-based framing (magic + type + length)
- Serialize/deserialize messages via big-endian streams (AIStreamBE/AOStreamBE)
- Pump socket data through receive state machines (header ΓåÆ body) and send state machines (header ΓåÆ body)
- Register and dispatch incoming messages to type-specific handlers via MessageDispatcher
- Deserialize received UninflatedMessage buffers into typed Message objects using MessageInflater prototypes
- Support synchronous receive with overall and inactivity timeouts
- Batch-flush outgoing messages across channels with activity timestamp tracking
- Accept incoming connections via CommunicationsChannelFactory
- Manage per-channel application state via Memento pattern

**Key dependencies (other subsystems):**
- **SDL_net**: TCP socket operations (`SDLNet_TCP_*`, `SDLNet_CheckSockets`)
- **SDL**: Timing (`SDL_GetTicks`), byte swapping (`SDL_SwapBE16`)
- **AStream**: Big-endian serialization (AIStreamBE, AOStreamBE)
- **Game engine**: Message type definitions and UninflatedMessage protocol

---

### XML

**Purpose:**
Orchestrates parsing of game configuration files (MML-derived markup) using the Expat C parser library. Implements a hierarchical tree of element-specific parsers that convert XML attributes into game engine data structures during engine initialization and level loading.

**Key directories / files:**
- Parser lifecycle: `XML_Configure.cpp/h`, `XML_ElementParser.cpp/h`
- Data handling: `XML_DataBlock.cpp/h`
- File I/O: `XML_Loader_SDL.cpp/h`
- Element parsers: `ColorParser.cpp/h`, `DamageParser.cpp/h`, `ShapesParser.cpp/h`, `XML_LevelScript.cpp/h`
- Initialization: `XML_MakeRoot.cpp`, `XML_ParseTreeRoot.h`

**Key responsibilities:**
- Manage Expat parser creation, lifecycle, and attribute/text callbacks
- Maintain element parser stack during XML traversal with semantic interpretation routing
- Parse and validate typed XML attribute values (integers with bounds, floats, booleans, strings)
- Convert float color channels and fixed-point scale values to engine-native formats
- Load XML configuration files from disk (FileSpecifier) and filter backup/script files
- Report three-tier error hierarchy: read errors (fatal), XML parse errors (with line numbers), interpretation errors (with logging throttle)
- Initialize parser tree and reset MML-configured values to hard-coded defaults
- Execute level-specific scripts (MML, Lua, music, load screens) at appropriate times
- UTF-8 to ASCII/Mac Roman conversion for legacy encoding support

**Key dependencies (other subsystems):**
- **Expat**: C XML parser library
- **File I/O**: FileSpecifier, OpenedFile, DirectorySpecifier
- **Music**: FadeOut, level music management
- **OGL_LoadScreen**: End-game screen configuration (conditional)
- **Lua**: Script loader (conditional)
- **Logging system**: Error reporting and anomaly throttling
- **Game engine types**: Color, damage, shape definitions

---

## Key Runtime Flows

### Initialization

1. **XML configuration load** (engine startup):
   - `SetupParseTree()` constructs the global parser hierarchy (~30 subsystem parsers)
   - `XML_Loader_SDL::ParseDirectory()` or `ParseFile()` loads configuration files
   - Expat parser feeds data to element parser tree; hierarchy navigation routes semantic interpretation
   - `ResetAllMMLValues()` restores MML-configured values to engine defaults
   - Configuration errors reported via logging with throttle; parse aborts if threshold exceeded

2. **Sound subsystem**:
   - `SoundManager::Initialize()` opens SDL audio device with configured sample rate and bit depth
   - `Mixer` singleton initialized with SDL audio callback context
   - Built-in `.snd2` sound definitions loaded and parsed
   - Music playback system initialized

3. **RenderOther subsystem**:
   - Motion sensor initialized
   - Font system initialized; platform-specific fonts (bitmap/TrueType) loaded
   - SDL rendering context created
   - HUD buffers allocated
   - Resource fonts loaded from engine resources

4. **TCPMess subsystem** (if networking enabled):
   - Implicit initialization of socket infrastructure via SDL_net
   - MessageDispatcher and MessageInflater singletons ready for message routing

### Per-frame / Main Loop

1. **Game update phase**:
   - RenderOther: `update_everything()` called to update HUD dirty elements (weapons, ammo, shield, oxygen, inventory)
   - Chase camera position updated once per tick with spring physics
   - Screen fades advanced based on elapsed ticks
   - View effects (FOV, fold transitions) interpolated toward target values

2. **Network I/O (if enabled)**:
   - TCPMess: Each CommunicationsChannel pumps I/O via `SDLNet_CheckSockets()`
   - Incoming UninflatedMessage buffers deserialized through registered MessageInflater prototypes
   - Typed Message instances dispatched to registered handlers via MessageDispatcher
   - Outgoing message queues batch-flushed across all channels

3. **Audio callback (asynchronous, SDL-driven)**:
   - Mixer callback executes in SDL audio context (tight timing)
   - Queries `Music::instance()->FillBuffer()` for music data
   - Dequeues network microphone audio if available
   - Performs real-time mixing from all active channels with format conversion (8/16-bit, mono/stereo, endianness)
   - Applies master volume and per-channel muting
   - Music buffer refilled from active StreamDecoder during callback

4. **Rendering composition**:
   - Core rendering loop routes to `screen_sdl.cpp:render_screen()`
   - Renders world view via hardware (OpenGL) or software rasterization
   - Composites game world, HUD layer, overhead map, terminal interface, crosshairs into final framebuffer
   - Swaps buffers or updates SDL display surface

### Shutdown

1. **RenderOther**: Rendering resources deallocated; HUD buffers freed; OpenGL context destroyed
2. **Sound**: `SoundManager::Shutdown()` closes SDL audio device; unloads sound definitions; clears replacement sounds; stops active playback
3. **TCPMess**: Socket connections closed implicitly
4. **XML**: Parser tree cleanup implicit (one-time parse, configuration retained)

---

## Data & Control Boundaries

**Global state and singletons:**
- **RenderOther**: HUD_Buffer, motion_sensor state, screen mode; accessed via screen.h APIs
- **Sound**: SoundManager, Mixer, Music singletons; accessed via SoundManager.h APIs
- **TCPMess**: MessageDispatcher singleton; routes incoming messages to registered handlers
- **XML**: RootParser global; parser tree created at startup and reused across level loads

**Resource ownership and lifetimes:**
- SDL audio device context: Created at Sound initialization, destroyed at shutdown; accessed from audio callback
- HUD buffers: Allocated once during RenderOther initialization; reused and marked dirty per-frame
- Parser tree: Created at startup via `SetupParseTree()`; reused for configuration and level script loads
- Socket connections: Created on demand via CommunicationsChannelFactory; destroyed on disconnect
- Decoded audio streams: Managed per-sound via StreamDecoder abstraction; memory-efficient streaming without full load into RAM

**Synchronization boundaries:**
- Audio callback (Mixer): Real-time context with strict timing constraints; uses lock/unlock for channel state changes
- Non-blocking socket I/O: State machines transition per frame; no blocking operations in game loop
- XML parse: Synchronous operation during initialization and level load; blocking until complete
- Message dispatch: Synchronous handler invocation per incoming message; handlers must not block

**Cross-subsystem data flow:**
- RenderOther ΓåÉ Sound: Terminal audio playback triggers, HUD sound effects
- RenderOther ΓåÉ XML: Font definitions, HUD layout, motion sensor parameters
- Sound ΓåÉ Game world: Listener position (via callback), obstruction queries (via callback), ambient sources (enumeration callback)
- TCPMess ΓåÉ Network layer: Socket state changes, incoming message buffers
- XML ΓåÆ All subsystems: Configuration data (colors, shapes, damage, music, scripts) drives subsystem initialization

---

## Notable Risks / Hotspots

1. **Multi-platform rendering abstraction**: RenderOther bridges OpenGL and SDL rasterization paths with platform-specific code paths and pixel format conversions (ARGB 1555/8888). Platform conditionals throughout; incorrect format assumptions can cause visual artifacts or crashes.

2. **Audio mixing format conversion**: Real-time format conversion (8/16-bit, mono/stereo, signed/unsigned, little/big-endian) performed during SDL audio callback. Tight timing constraints and endianness bugs can cause audio glitches or performance degradation.

3. **Non-blocking socket I/O state machines**: TCPMess maintains header ΓåÆ body state transitions for both receive and send. Incorrect state advancement or incomplete buffer management can cause message loss or corruption.

4. **XML parse error recovery**: Three-tier error hierarchy with throttling; parse can abort on interpretation error threshold. Incomplete or malformed configuration files may leave engine in inconsistent state.

5. **System 7 sound format compatibility**: Big-endian header parsing with multiple permutation support. Endianness bugs or incorrect permutation selection can cause audio corruption or crashes.

6. **Chase camera collision detection**: Raycast collisions against world geometry per frame; geometry queries can impact frame rate if geometry is complex.

7. **Memory constraints on PSP**: Limited RAM (32 MB on original PSP, 64 MB on slim). Audio buffer sizes, HUD buffer allocation, and parser tree depth must be carefully managed to avoid out-of-memory conditions.

8. **SDL audio callback context**: Real-time callback requires lock-free or minimally-locking operations. Unprotected state changes in mixer context can cause race conditions or audio dropout.
