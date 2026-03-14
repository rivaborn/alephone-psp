# Source_Files/RenderOther/ChaseCam.cpp

## File Purpose
Implements a third-person chase camera system for Marathon, providing a Halo-like behind-the-back view. The camera follows the player with configurable springiness and inertia, and respects level geometry to avoid clipping through walls.

## Core Responsibilities
- Maintain chase camera state (active/inactive, reset flag)
- Store and manage camera position history for inertia calculations
- Apply spring/damping physics to smooth camera movement
- Raycast camera position against level geometry (walls, floors, ceilings)
- Provide interface for enabling/disabling camera and switching horizontal offset
- Update camera position once per game tick

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `world_point3d` | typedef | 3D position in world space |
| `ChaseCamData` | struct | Configuration (offset distances, damping, spring constants) |
| `polygon_data` | struct | Map polygon with height bounds and adjacent geometry |
| `line_data` | struct | Map line segment with endpoint references |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_ChaseCam_IsActive` | bool | static | Current activation state |
| `_ChaseCam_IsReset` | bool | static | Flag to trigger position history reset on next update |
| `CC_Position` | world_point3d | static | Current camera position |
| `CC_Position_1`, `CC_Position_2` | world_point3d | static | Previous two camera positions (for inertia) |
| `CC_Polygon`, `CC_Yaw`, `CC_Pitch` | short, angle, angle | static | Current camera polygon index and orientation angles |

## Key Functions / Methods

### ChaseCam_CanExist
- Signature: `bool ChaseCam_CanExist()`
- Purpose: Check if chase cam feature is available (not disabled in config)
- Inputs: None
- Outputs/Return: True if chase cam can be activated
- Side effects: None
- Calls: `GetChaseCamData()`, `TEST_FLAG()` macro
- Notes: Used to avoid loading player sprites if feature is disabled

### ChaseCam_IsActive
- Signature: `bool ChaseCam_IsActive()`
- Purpose: Query current active state with network and config guards
- Inputs: None
- Outputs/Return: True if camera is currently rendering
- Side effects: None
- Calls: `NetAllowBehindview()`, `ChaseCam_CanExist()`
- Notes: Returns false in single-player network mode or if feature disabled

### ChaseCam_SetActive
- Signature: `bool ChaseCam_SetActive(bool NewState)`
- Purpose: Enable or disable the chase camera
- Inputs: Desired state
- Outputs/Return: New state (may differ from input if invalid)
- Side effects: Updates `_ChaseCam_IsActive`
- Calls: `NetAllowBehindview()`, `ChaseCam_CanExist()`

### ChaseCam_Initialize
- Signature: `bool ChaseCam_Initialize()`
- Purpose: Initialize camera for a new game/level
- Inputs: None
- Outputs/Return: Success flag
- Side effects: Calls `ChaseCam_Reset()`, sets initial active state from config
- Calls: `ChaseCam_CanExist()`, `ChaseCam_Reset()`, `GetChaseCamData()`, `TEST_FLAG()`

### ChaseCam_Reset
- Signature: `bool ChaseCam_Reset()`
- Purpose: Reset camera history when entering level, reviving, or teleporting
- Inputs: None
- Outputs/Return: Success flag
- Side effects: Sets `_ChaseCam_IsReset` flag
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`
- Notes: Flag triggers position history reset on next `ChaseCam_Update()`

### ChaseCam_SwitchSides
- Signature: `bool ChaseCam_SwitchSides()`
- Purpose: Negate horizontal offset to switch camera to opposite side
- Inputs: None
- Outputs/Return: Success flag
- Side effects: Negates `ChaseCam.Rightward` configuration value
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`, `GetChaseCamData()`

### ChaseCam_GetPosition
- Signature: `bool ChaseCam_GetPosition(world_point3d &position, short &polygon_index, angle &yaw, angle &pitch)`
- Purpose: Retrieve current camera position and orientation for rendering
- Inputs: References to output position/orientation variables
- Outputs/Return: True if camera is active (output parameters updated); false otherwise, leaving outputs unchanged
- Side effects: None
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`
- Notes: Called by renderer; does not update camera state

### CC_PosUpdate
- Signature: `static int CC_PosUpdate(float Damping, float Spring, short x0, short x1, short x2)`
- Purpose: Apply spring-damper physics to a single position coordinate
- Inputs: Damping coefficient, spring coefficient, current/previous/previous-previous position
- Outputs/Return: New damped position
- Side effects: None
- Calls: None
- Notes: Implements second-order linear difference equation; x0=current, x1=t-1, x2=t-2

### ShootForTargetPoint
- Signature: `static void ShootForTargetPoint(bool ThroughWalls, world_point3d& StartPosition, world_point3d& EndPosition, short& Polygon)`
- Purpose: Raycast from start to end position, clipping against level geometry
- Inputs: Through-walls flag, start position, target end position, start polygon index
- Outputs/Return: Modifies EndPosition and Polygon (final clipped position and polygon)
- Side effects: None (input/output references modified)
- Calls: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, geometry-query functions (`find_floor_or_ceiling_intersection()`, `find_line_crossed_leaving_polygon()`, `find_line_intersection()`, `find_adjacent_polygon()`)
- Notes: Handles wraparound of short integer when crossing map boundaries; clipping depends on `ThroughWalls` flag

### ChaseCam_Update
- Signature: `bool ChaseCam_Update()`
- Purpose: Update camera position for one game tick with physics simulation
- Inputs: None (reads `current_player` and config)
- Outputs/Return: Success flag
- Side effects: Updates `CC_Position`, `CC_Position_1`, `CC_Position_2`, `CC_Polygon`, `CC_Yaw`, `CC_Pitch`, `_ChaseCam_IsReset`
- Calls: `ChaseCam_CanExist()`, `ChaseCam_IsActive()`, `GetChaseCamData()`, `translate_point3d()`, `translate_point2d()`, `CC_PosUpdate()`, `ShootForTargetPoint()`, `TEST_FLAG()`, `NORMALIZE_ANGLE()`
- Notes: **Main frame update function.** Shifts position history, calculates target offset from player, optionally applies inertia via spring-damper, raycasts against geometry, resets history if needed.

## Control Flow Notes
**Initialization:** `ChaseCam_Initialize()` called when level loads, resets camera if configured to start active.

**Per-Frame:** `ChaseCam_Update()` executes each game tick, maintaining smooth motion with springiness. Position queried by rendering system via `ChaseCam_GetPosition()`.

**Transitions:** `ChaseCam_Reset()` triggered on level entry, teleport, or revival; defers position-history reset to next update to preserve inertia setup.

**Network:** Disabled if `NetAllowBehindview()` returns false (multiplayer cheat restrictions).

## External Dependencies
- **Map geometry:** `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, geometry query functions
- **Player state:** `current_player` global (camera location, orientation, polygon)
- **Configuration:** `GetChaseCamData()` (preferences), `TEST_FLAG()` macro (bit ops)
- **Network:** `NetAllowBehindview()` (multiplayer restrictions)
- **Math/geometry:** `translate_point3d()`, `translate_point2d()`, `NORMALIZE_ANGLE()` macro
- **Includes:** `cseries.h`, `map.h`, `player.h`, `ChaseCam.h`, `network.h`, `<limits.h>`
