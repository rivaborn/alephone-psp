# Source_Files/GameWorld/items.h

## File Purpose
Header file defining the item system for the game world. Declares item type enumerations, item management functions, and XML configuration support. Items include weapons, ammunition, powerups, and interactive balls used in multiplayer modes.

## Core Responsibilities
- Define item class types (weapons, ammunition, powerups, balls, etc.) and specific item IDs
- Create and place items in the world (`new_item`, `new_item_in_random_location`)
- Manage player inventory and item pickup logic (`try_and_add_player_item`, `swipe_nearby_items`)
- Query item metadata and properties (`get_item_kind`, `get_item_shape`, `get_item_definition_external`)
- Validate items in current environment and count inventory capacity
- Handle item interaction triggers (`trigger_nearby_items`)
- Track ball ownership for multiplayer game modes (`find_player_ball_color`)
- Provide frame-based animation and initialization via callbacks
- Support XML-driven configuration of item properties

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| (item type enum) | enum | Classifies items into 7 categories: weapon, ammunition, powerup, item, weapon_powerup, ball, network_statistics |
| (item ID enum) | enum | Enumerates 50+ specific item instances (knives, magnums, grenades, alien weapons, powerups, balls, SMG, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### new_item
- Signature: `short new_item(struct object_location *location, short item_type)`
- Purpose: Create and place a new item instance at a specific world location
- Inputs: World location struct; item type ID
- Outputs/Return: Item index or error code
- Calls: (implementation in items.c)

### try_and_add_player_item
- Signature: `bool try_and_add_player_item(short player_index, short type)`
- Purpose: Attempt to add an item to a player's inventory; returns false if inventory full
- Inputs: Player index; item type
- Outputs/Return: Success flag
- Calls: (implementation in items.c)

### calculate_player_item_array
- Signature: `void calculate_player_item_array(short player_index, short type, short *items, short *counts, short *array_count)`
- Purpose: Build arrays of item IDs and counts for display/management UI
- Inputs: Player index; item type filter; output pointers
- Outputs/Return: Populates items, counts, array_count
- Calls: (implementation in items.c)

### get_item_definition_external
- Signature: `struct item_definition *get_item_definition_external(const short type)`
- Purpose: Retrieve metadata/configuration struct for an item type (exposed for scripting/XML)
- Inputs: Item type ID
- Outputs/Return: Pointer to item_definition struct
- Calls: (implementation in items.c)

### find_player_ball_color
- Signature: `short find_player_ball_color(short player_index)`
- Purpose: Determine which colored ball (if any) a player is carrying; used in ball-game modes
- Inputs: Player index
- Outputs/Return: Ball color constant or NONE
- Calls: (implementation in items.c)

### initialize_items / animate_items
- Signature: `void initialize_items(void); void animate_items(void);`
- Purpose: Per-frame and initialization callbacks for item animation and state updates
- Calls: (implementation in items.c)

**Trivial helpers** (summarized): `get_item_kind()`, `get_item_shape()`, `get_header_name()`, `get_item_name()`, `item_valid_in_current_environment()`, `trigger_nearby_items()`, `unretrieved_items_on_map()`, `count_inventory_lines()`, `swipe_nearby_items()`, `mark_item_collections()` ΓÇö inventory queries, name lookups, and validation checks.

## Control Flow Notes
- **Initialization**: `initialize_items()` called at game startup
- **Per-frame**: `animate_items()` called each frame for visual/state updates
- **Pickup/Interaction**: `try_and_add_player_item()` and `trigger_nearby_items()` invoked on collision/proximity
- **Inventory Display**: `calculate_player_item_array()` and count functions drive UI rendering
- Items marked for collection via `mark_item_collections()` during level load/save

## External Dependencies
- **XML_ElementParser.h**: C++ XML parsing framework for configuration
- **object_location**: Structure defined elsewhere (world position/polygon)
- **item_definition**: Structure defined elsewhere (item metadata, damage, animation, etc.)
- Callers in: game_window.c, scenery system, player/inventory systems, collision handlers
