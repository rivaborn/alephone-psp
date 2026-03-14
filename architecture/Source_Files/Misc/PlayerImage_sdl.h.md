# Source_Files/Misc/PlayerImage_sdl.h

## File Purpose
Defines `PlayerImage`, a class that manages the visual state and rendering of player character sprites in SDL. It separates legs and torso rendering with independent appearance parameters, using lazy evaluation to defer expensive graphics loading until needed.

## Core Responsibilities
- Manage player visual state (view angle, color, animation action/frame, brightness, size) separately for legs and torso
- Lazy-load and cache SDL graphics surfaces based on current visual state
- Track dirty state to minimize recomputation of expensive graphics operations
- Provide query methods to check validity of loaded drawing data before rendering
- Handle SDL collection resource marking/unmarking across all instances via static object counter

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PlayerImage | class | Main class managing player sprite appearance and rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sNumOutstandingObjects | int16 | static class member | Count of active PlayerImage instances; used to coordinate collection marking/unmarking |

## Key Functions / Methods

### Constructor
- Signature: `PlayerImage()`
- Purpose: Initialize player image with default/random visual parameters; set up lazy-load infrastructure
- Inputs: None
- Outputs/Return: Constructed object with all state initialized
- Side effects: Increments `sNumOutstandingObjects` via `objectCreated()`; allocates object
- Calls: `objectCreated()`
- Notes: All state initialized to NONE (randomly chosen when needed); no graphics loaded until update called

### setLegsView / setTorsoView / setView
- Signature: `void setLegsView(int16 inView)`, `void setTorsoView(int16 inView)`, `void setView(int16 inView)`
- Purpose: Set which direction (0ΓÇô7) legs/torso face; 0 = toward viewer, increments rotate player right 45┬░
- Inputs: View angle 0ΓÇô7; NONE (ΓÇô1) for random
- Outputs/Return: None
- Side effects: Sets dirty flag if value changed
- Calls: None (setView delegates to setLegsView/setTorsoView)
- Notes: Setter pattern: only marks dirty if value actually changed, avoiding unnecessary updates

### setLegsColor / setTorsoColor / setColor / setPlayerColor / setTeamColor
- Signature: `void setLegsColor(int16 inColor)`, `void setTorsoColor(int16 inColor)`, etc.
- Purpose: Set color (0ΓÇô7, standard player/team colors); legs typically = team color, torso = player color
- Inputs: Color 0ΓÇô7; NONE for random
- Outputs/Return: None
- Side effects: Sets dirty flag if value changed
- Calls: Delegated methods call primary setters
- Notes: Legs and torso can have independent colors

### setLegsAction / setTorsoAction / setPseudoWeapon / setLegsFrame / setTorsoFrame
- Signature: Various `void set*(...int16 in...)` methods
- Purpose: Set animation action (from player.h / weapons.h), pseudo-weapon, and frame number
- Inputs: Action/weapon/frame enum values; NONE for random
- Outputs/Return: None
- Side effects: Set dirty flag if value changed
- Calls: None
- Notes: Legs and torso animation states independent; frame indexing depends on action

### setLegsBrightness / setTorsoBrightness / setBrightness
- Signature: `void setLegsBrightness(float inBrightness)`, etc.
- Purpose: Set lighting intensity (0.0ΓÇô1.0) for legs/torso
- Inputs: Float 0.0ΓÇô1.0
- Outputs/Return: None
- Side effects: Sets dirty flag if value changed
- Calls: None
- Notes: Allows independent brightness for each body part

### setTiny
- Signature: `void setTiny(bool inTiny)`
- Purpose: Enable quarter-sized rendering
- Inputs: Boolean
- Outputs/Return: None
- Side effects: Sets both `mLegsDirty` and `mTorsoDirty` if value changed
- Calls: None
- Notes: Affects both legs and torso

### updateLegsDrawingInfo / updateTorsoDrawingInfo / updateDrawingInfo
- Signature: `void updateLegsDrawingInfo()`, `void updateTorsoDrawingInfo()`, `void updateDrawingInfo()`
- Purpose: Synchronize cached SDL surfaces and metadata with current visual state
- Inputs: None (uses object's state members)
- Outputs/Return: None
- Side effects: Loads/interprets graphics from collections; may allocate `mLegsData`/`mTorsoData`; sets `mLegsValid`/`mTorsoValid` flags
- Calls: (not defined in header; implementation in .cpp)
- Notes: Expensive operations; should be called on-demand, not every frame; `updateDrawingInfo()` is a batch dispatcher

### canDrawLegs / canDrawTorso / canDrawSomething / canDrawPlayer
- Signature: `bool canDrawLegs()`, `bool canDrawTorso()`, `bool canDrawSomething()`, `bool canDrawPlayer()`
- Purpose: Check if valid graphics data is available for rendering; returns validity flag after ensuring data is up-to-date
- Inputs: None
- Outputs/Return: Boolean (true if graphics loaded and valid)
- Side effects: Calls `updateDrawingInfo()` to synchronize state
- Calls: `updateDrawingInfo()`
- Notes: `canDrawPlayer()` requires both legs AND torso valid; `canDrawSomething()` requires at least one

### getTeam
- Signature: `int16 getTeam()`
- Purpose: Query the team color (legs color)
- Inputs: None
- Outputs/Return: Team color value (0ΓÇô7) or NONE
- Side effects: Calls `updateDrawingInfo()` to ensure state is current
- Calls: `updateDrawingInfo()`
- Notes: Comment suggests more query methods should exist

### drawAt
- Signature: `void drawAt(SDL_Surface* inSurface, int16 inX, int16 inY)`
- Purpose: Render the player image to an SDL surface at specified screen coordinates
- Inputs: Target SDL_Surface pointer, X and Y pixel coordinates
- Outputs/Return: None
- Side effects: Calls `updateDrawingInfo()` to ensure graphics are loaded; blits to target surface
- Calls: `updateDrawingInfo()` and SDL blit operations (not defined in header)
- Notes: Will skip rendering any parts that are invalid (mLegsValid/mTorsoValid = false)

## Control Flow Notes
This class operates in a **render/UI context**. Typical flow:
1. **Setup phase**: Create `PlayerImage`, call setters to configure appearance
2. **Lazy load phase**: First query or draw triggers `updateDrawingInfo()`, which loads graphics from game collections
3. **Rendering phase**: `drawAt()` blits pre-loaded surfaces to screen at desired position
4. **Teardown**: Destructor frees allocated `mLegsData`/`mTorsoData`, decrements `sNumOutstandingObjects`

The design defers expensive operations (collection loading, graphics interpretation) until actually needed. State changes merely flag members as dirty; actual updates happen on-demand before drawing.

## External Dependencies
- `SDL.h` ΓÇô SDL_Surface, SDL_Rect types
- `cseries.h` ΓÇô platform abstraction layer (int16, byte types, macros)
- Implicit: `player.h` ΓÇô leg action enums (e.g., `_player_running`)
- Implicit: `weapons.h` ΓÇô torso action enums (e.g., `_shape_weapon_firing`), weapon enums (e.g., `_weapon_missile_launcher`)
