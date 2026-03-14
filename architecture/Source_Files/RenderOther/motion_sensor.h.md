# Source_Files/RenderOther/motion_sensor.h

## File Purpose
Header file declaring the motion sensor system interface for the Aleph One game engine. Provides functions to initialize, manage, and query the in-game motion tracker that displays enemy/object detection, plus XML configuration support.

## Core Responsibilities
- Initialize motion sensor with shape descriptors for mounts, aliens, friendly units, enemies, and compass
- Reset motion sensor state per monster/entity
- Query motion sensor state changes (for dirty-flag/update optimization)
- Dynamically adjust motion sensor detection range
- Provide XML element parser for motion sensor configuration

## Key Types / Data Structures
None defined here.

| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef (defined elsewhere) | Identifier for shapes used in motion sensor visualization |
| `XML_ElementParser` | class (XML_ElementParser.h) | Base class for parsing motion sensor XML elements |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_motion_sensor
- **Signature:** `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor alien, shape_descriptor _friend, shape_descriptor enemy, shape_descriptor network_compass, short side_length)`
- **Purpose:** Initialize motion sensor graphics and state with shape descriptors for all entity types and the compass element.
- **Inputs:** Six shape descriptors (mount variants, alien, friendly, enemy, compass), side length (likely motion sensor window size in pixels).
- **Outputs/Return:** None.
- **Side effects:** Likely allocates motion sensor state, stores shape references for rendering.
- **Calls:** (Not inferable from header.)
- **Notes:** Called once at startup or map load.

### reset_motion_sensor
- **Signature:** `void reset_motion_sensor(short monster_index)`
- **Purpose:** Clear or reinitialize motion sensor state for a specific entity (monster).
- **Inputs:** Entity index.
- **Outputs/Return:** None.
- **Side effects:** Clears cached detection state for the entity.
- **Calls:** (Not inferable.)
- **Notes:** Invoked when entity is destroyed or despawned.

### motion_sensor_has_changed
- **Signature:** `bool motion_sensor_has_changed(void)`
- **Purpose:** Check if motion sensor display needs update (entity positions changed).
- **Inputs:** None.
- **Outputs/Return:** `true` if any tracked entity moved; `false` if static.
- **Side effects:** None visible; likely resets internal dirty flag after return.
- **Calls:** (Not inferable.)
- **Notes:** Optimization flag for skipping re-render of unchanged sensor.

### adjust_motion_sensor_range
- **Signature:** `void adjust_motion_sensor_range(void)`
- **Purpose:** Dynamically update motion sensor detection radius (e.g., powerup effect, difficulty scaling).
- **Inputs:** None (reads global game state).
- **Outputs/Return:** None.
- **Side effects:** Updates motion sensor range parameter.
- **Calls:** (Not inferable.)
- **Notes:** Called during frame update or when range-affecting state changes.

### MotionSensor_GetParser
- **Signature:** `XML_ElementParser *MotionSensor_GetParser()`
- **Purpose:** Retrieve XML parser for motion sensor configuration elements.
- **Inputs:** None.
- **Outputs/Return:** Pointer to `XML_ElementParser` (element name "motion_sensor").
- **Side effects:** None.
- **Calls:** (Not inferable.)
- **Notes:** Part of engine's XML configuration framework (Loren Petrich addition, May 2000).

## Control Flow Notes
Fits into the **frame/update** phase: `motion_sensor_has_changed()` gates re-rendering; `adjust_motion_sensor_range()` updates game rules. Initialization occurs at map/scenario load. XML parsing happens during engine startup for static configuration.

## External Dependencies
- **XML_ElementParser.h** ΓÇô Base class for XML parsing framework.
- **shape_descriptor** ΓÇô Type defined elsewhere; references shape data for rendering motion sensor icons.
- Implementation in `motion_sensor.c` (referenced in comment).
