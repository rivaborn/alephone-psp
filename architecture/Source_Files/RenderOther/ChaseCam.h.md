# Source_Files/RenderOther/ChaseCam.h

## File Purpose
Interface for a third-person chase camera system that follows the player, similar to Halo's implementation. Provides functions to configure, initialize, update, and query the state of a follow camera that maintains a dynamic offset from the player's position and orientation.

## Core Responsibilities
- Define chase camera configuration parameters (distance offsets, physics tuning, visibility flags)
- Initialize and manage chase camera lifecycle (startup, per-level reset, shutdown)
- Update camera position and orientation each frame using physics simulation
- Track active/inactive state and support toggling the camera on/off
- Query final camera position and viewing angles for rendering
- Support side-switching behavior when camera is offset laterally

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ChaseCamData` | struct | Configuration bundle for chase camera behavior: offset distances (Behind, Upward, Rightward), physics parameters (Damping, Spring), opacity, and behavioral flags |
| Chase-cam flags enum | enum | Bitflags controlling behavior: `_ChaseCam_OnWhenEntering` (auto-enable when entering level), `_ChaseCam_NeverActive` (disable entirely), `_ChaseCam_ThroughWalls` (render through geometry) |

## Global / File-Static State
None (implementation details deferred to `.c` files; state managed via accessors).

## Key Functions / Methods

### ChaseCam_IsActive / ChaseCam_SetActive
- Signature: `bool ChaseCam_IsActive()`, `bool ChaseCam_SetActive(bool NewState)`
- Purpose: Query or toggle whether the chase camera is currently rendering
- Inputs: `NewState` (desired active state)
- Outputs/Return: `bool` (current/new active state)
- Side effects: Sets internal chase-cam state
- Notes: Both return state as `bool` (true=active, false=inactive)

### ChaseCam_Initialize / ChaseCam_Reset
- Signature: `bool ChaseCam_Initialize()`, `bool ChaseCam_Reset()`
- Purpose: Initialize chase camera for a new game; reset it when entering a level, reviving, or teleporting
- Outputs/Return: `bool` (success status)
- Side effects: Allocates/resets camera position, velocity, and physics accumulators
- Notes: Called at different points in the game lifecycle; `Reset()` maintains game state but resets camera geometry

### ChaseCam_Update
- Signature: `bool ChaseCam_Update()`
- Purpose: Advance camera physics simulation over one game tick
- Outputs/Return: `bool` (active state, for efficiency)
- Side effects: Updates internal position/velocity based on spring and damping parameters
- Notes: Must be called every tick for correct physics; otherwise camera will not smooth-follow the player

### ChaseCam_GetPosition
- Signature: `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- Purpose: Retrieve the final camera view position and orientation for rendering
- Outputs/Return: Fills `position` (3D world coordinates), `polygon_index` (BSP cell), `yaw`/`pitch` (viewing angles); returns active state
- Side effects: None (read-only query)
- Notes: Only valid if camera is active; all outputs are references that are not modified if inactive

### ChaseCam_SwitchSides / ChaseCam_CanExist
- Signature: `bool ChaseCam_SwitchSides()`, `bool ChaseCam_CanExist()`
- Purpose: Toggle lateral offset direction when camera is to one side; check if chase camera assets can be loaded
- Outputs/Return: `bool` (state or feasibility)
- Calls: `ChaseCam_CanExist()` used to avoid loading player sprites if not needed
- Notes: `SwitchSides()` implies continuous lateral offset rather than always-behind positioning

## Control Flow Notes
- **Init**: `ChaseCam_Initialize()` called once at game start; `ChaseCam_CanExist()` checked before loading player sprite assets
- **Per-frame**: `ChaseCam_Update()` (physics), then `ChaseCam_GetPosition()` (query for rendering)
- **Events**: `ChaseCam_Reset()` on level entry, revival, or teleport; `ChaseCam_SetActive()` toggles user control
- Integration point: Rendering passes call `GetPosition()` to obtain camera pose after `Update()` completes

## External Dependencies
- **`world.h`**: Core type definitions (`world_point3d`, `angle`, `world_distance`) and coordinate system macros
- **"PlayerDialogs.c"**: Implements `Configure_ChaseCam()` (settings UI)
- **"preferences.c"**: Implements `GetChaseCamData()` (persistent configuration loading)
