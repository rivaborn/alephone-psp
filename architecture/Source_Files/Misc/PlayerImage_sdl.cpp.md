# Source_Files/Misc/PlayerImage_sdl.cpp

## File Purpose

SDL-based implementation of player character image rendering for the Aleph One game engine. Manages caching and drawing of pre-rendered 2D player sprites (legs and torso separately), with automatic collection loading and validation of animation frame data.

## Core Responsibilities

- **Resource lifecycle**: Allocate/free SDL surfaces and pixel buffers for leg and torso imagery  
- **Lazy evaluation**: Update cached drawing data only when dirty flags are set  
- **Parameter validation**: Handle invalid/missing state (NONE values) via random selection with retry logic  
- **Collection management**: Track active PlayerImage instances to mark/unload shape collections  
- **Sprite drawing**: Composite two surfaces (legs + torso) to an SDL target at specified coordinates  
- **Animation support**: Resolve action/view/frame indices through shape definition hierarchy  

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `PlayerImage` | class | Main player sprite renderer; member variables in header |
| `player_shape_definitions` | struct | Maps actions/weapons to animation indices (from player.h) |
| `shape_animation_data` | struct | Metadata: frame count, view count, ticks per frame (from interface.h) |
| `low_level_shape_definition` | struct | Individual sprite: origin, keypoint, bitmap index (from collection_definition.h) |
| `SDL_Surface` | opaque | Source/target raster surfaces (SDL library) |
| `SDL_Rect` | struct | Rectangular blit region and positioning |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bit_depth` | extern short | global | Display bit depth; 8-bit skips shape loading (line ~136, ~226) |
| `sNumOutstandingObjects` | static int16 | class | Reference count of active PlayerImage objects; triggers collection load/unload |

## Key Functions / Methods

### ~PlayerImage (destructor)
- **Signature:** `~PlayerImage()`  
- **Purpose:** Release SDL surfaces and pixel data; decrement global object count  
- **Inputs:** None  
- **Outputs/Return:** None  
- **Side effects:** Frees mLegsSurface, mTorsoSurface, mLegsData, mTorsoData; calls objectDestroyed()  
- **Calls:** SDL_FreeSurface(), free(), objectDestroyed()  

### setRandomFlatteringView
- **Signature:** `void setRandomFlatteringView()`  
- **Purpose:** Set both legs and torso to a random view angle, avoiding rear-facing poses (views 3ΓÇô5 are "butt shots")  
- **Inputs:** None  
- **Outputs/Return:** None  
- **Side effects:** Updates mLegsView and mTorsoView; sets mLegsDirty, mTorsoDirty  
- **Calls:** local_random(), setView()  

### updateLegsDrawingInfo
- **Signature:** `void updateLegsDrawingInfo()`  
- **Purpose:** Resolve leg animation parameters, fetch SDL surface, cache rect/brightness metadata  
- **Inputs:** Member state (mLegsAction, mLegsView, mLegsFrame, mLegsColor, mLegsBrightness); uses mLegsDirty  
- **Outputs/Return:** Updates mLegsSurface, mLegsData, mLegsRect; sets mLegsValid  
- **Side effects:** Allocates SDL surface; modifies member variables on success or failure  
- **Calls:**  
  - get_player_shape_definitions()  
  - get_shape_animation_data()  
  - get_low_level_shape_definition()  
  - get_shape_surface()  
  - local_random()  
- **Notes:**  
  - Retry loop (max 100 iterations) randomly re-picks parameters if any step fails  
  - Skip if bit_depth == 8 (line ~136)  
  - NONE (-1) values trigger random selection; invalid ranges abort  
  - Uses key_x/key_y for rect origin from shape definition  

### updateTorsoDrawingInfo
- **Signature:** `void updateTorsoDrawingInfo()`  
- **Purpose:** Resolve torso animation parameters (action, pseudo-weapon, view, frame), fetch surface  
- **Inputs:** mTorsoAction, mPseudoWeapon, mTorsoView, mTorsoFrame, mTorsoColor, mTorsoBrightness  
- **Outputs/Return:** Updates mTorsoSurface, mTorsoData, mTorsoRect; sets mTorsoValid  
- **Side effects:** Allocates SDL surface; modifies member variables  
- **Calls:**  
  - get_player_shape_definitions()  
  - get_shape_animation_data()  
  - get_low_level_shape_definition()  
  - get_shape_surface()  
  - local_random()  
- **Notes:**  
  - Similar retry loop to updateLegsDrawingInfo  
  - Switch on mTorsoAction selects torsos[], firing_torsos[], or charging_torsos[]  
  - Uses origin_x/y (not key_x/y) for rect origin  
  - Skip if bit_depth == 8  

### drawAt
- **Signature:** `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`  
- **Purpose:** Composite cached leg and torso sprites onto target surface at screen coordinates  
- **Inputs:** inSurface (target SDL surface), inX/inY (screen position)  
- **Outputs/Return:** None  
- **Side effects:** Modifies inSurface pixels; calls update routines if dirty  
- **Calls:** canDrawLegs(), canDrawTorso(), SDL_BlitSurface()  
- **Notes:** Does not validate inSurface; behavior undefined if NULL or invalid  

### objectCreated (static)
- **Signature:** `static void objectCreated()`  
- **Purpose:** Increment object count; on first object, mark/load the player shape collection  
- **Inputs:** None (uses static state)  
- **Outputs/Return:** None  
- **Side effects:** Increments sNumOutstandingObjects; may call mark_collection() and load_collections()  
- **Calls:** get_player_shape_definitions(), mark_collection(), load_collections()  
- **Notes:** Commented-out line suggests attempted duplicate-load guard  

### objectDestroyed (static)
- **Signature:** `static void objectDestroyed()`  
- **Purpose:** Decrement object count; unmark collection when last object is destroyed  
- **Inputs:** None (uses static state)  
- **Outputs/Return:** None  
- **Side effects:** Decrements sNumOutstandingObjects; may call mark_collection() to unload  
- **Calls:** get_player_shape_definitions(), mark_collection()  

## Control Flow Notes

**Initialization:** Constructor calls objectCreated() (in header); first instance loads collection.  

**Frame update:** Dirty flags trigger updateLegsDrawingInfo() / updateTorsoDrawingInfo() on first canDraw*/drawAt call (lazy).  

**Shutdown:** Destructor calls objectDestroyed(); last instance unmarks collection for unloading.  

**Retry logic:** Both update methods loop up to 100 times if random choices fail (NULL pointers, invalid indices), giving stochastic robustness.  

## External Dependencies

- **world.h**: `local_random()` for RNG  
- **player.h**: `NUMBER_OF_PLAYER_ACTIONS`, `player_shape_definitions`, `PLAYER_TORSO_*` constants  
- **interface.h**: `get_shape_animation_data()`, `get_shape_information()`, `mark_collection()`, `load_collections()`, collection/shape lookup  
- **shell.h**: `get_shape_surface()` (SDL surface generation from shape data)  
- **collection_definition.h**: `low_level_shape_definition` struct  
- **SDL library**: `SDL_FreeSurface()`, `SDL_BlitSurface()`, `SDL_Surface`, `SDL_Rect`
