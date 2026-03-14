# Source_Files/GameWorld/marathon2.cpp

## File Purpose
Core game world initialization and main update loop for the Marathon engine. Orchestrates all per-tick game state updates (player movement, monsters, projectiles, effects, platforms, etc.), manages level transitions, and implements client-side prediction for network multiplayer.

## Core Responsibilities
- Game world initialization (memory allocation, subsystem setup)
- Main game loop (`update_world()`) coordinating all entity updates
- Action queue management for input/Lua/prediction
- Level entry/exit and cleanup
- Game-state saving/restoration for client prediction
- Polygon trigger handling (monster activation, platform control, damage zones)
- Mission objective completion evaluation
- Damage calculation and difficulty scaling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ModifiableActionQueues` | class (external) | Queue of per-player action flags; supports enqueue/dequeue/peek operations |
| `player_data` | struct (external) | Player entity state; includes position, health, inventory, monster_index |
| `monster_data` | struct (external) | Monster entity state; includes position, target, vitality, action/mode |
| `object_data` | struct (external) | Generic world object (projectile, effect, scenery); animation and physics |
| `damage_definition` | struct (external) | Damage parameters: type, base, random, scale, flags for difficulty modifier |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GameQueue` | `ModifiableActionQueues*` | static | Intermediate action-queue buffer for transferring flags to the update loop |
| `sMostRecentFlagsForPlayer[MAXIMUM_NUMBER_OF_PLAYERS]` | `uint32[]` | static | Captured per-player action flags; used to extrapolate prediction |
| `sPredictedTicks` | `size_t` | static | Count of ticks simulated during prediction phase |
| `sPredictionWanted` | `bool` | static | Flag: whether to perform client-side prediction this frame |
| `sSavedPlayerData[MAXIMUM_NUMBER_OF_PLAYERS]` | `player_data[]` | static | Saved player state (position, health, etc.) before entering prediction |
| `sSavedPlayerMonsterData[MAXIMUM_NUMBER_OF_PLAYERS]` | `monster_data[]` | static | Saved monster (player body) state before prediction |
| `sSavedPlayerObjectData[MAXIMUM_NUMBER_OF_PLAYERS]` | `object_data[]` | static | Saved player object (physics body) before prediction |
| `sSavedPlayerParasiticObjectData[MAXIMUM_NUMBER_OF_PLAYERS]` | `object_data[]` | static | Saved attached/parasitic object state before prediction |

## Key Functions / Methods

### initialize_marathon
- Signature: `void initialize_marathon(void)`
- Purpose: One-time engine startup; allocates memory for all game subsystems and initializes core managers.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Allocates map, pathfinding, texture, and weapon memory; initializes rendering, items, scenery; creates GameQueue; calls `OGL_Initialize()` on OpenGL platforms.
- Calls: `build_trig_tables()`, `allocate_map_memory()`, `allocate_pathfinding_memory()`, `allocate_texture_tables()`, `initialize_weapon_manager()`, `initialize_game_window()`, `initialize_scenery()`, `initialize_items()`, `OGL_Initialize()`.
- Notes: Called once at engine startup; assumes subsystem headers are available.

### update_world
- Signature: `std::pair<bool, int16> update_world(void)`
- Purpose: Main game loop update; advances all game entities one or more ticks, then performs client-side prediction if enabled.
- Inputs: None (reads from `GameQueue`, Lua action queues, network time).
- Outputs/Return: `{didPredict || elapsed > 0, elapsedTicks}` ΓÇö whether redraw is needed and real tick count.
- Side effects: Increments `dynamic_world->tick_count`, decrements `game_time_remaining`, updates player/monster/projectile positions, fires polygon triggers, restores/saves prediction state, updates UI/fades.
- Calls: `NetProcessMessagesInGame()`, `overlay_queue_with_queue_into_queue()`, `GetRealActionQueues()`, `GetLuaActionQueues()`, `update_world_elements_one_tick()`, `exit_predictive_mode()`, `enter_predictive_mode()`, `update_interface()`, `update_fades()`, `check_recording_replaying()`, `game_timed_out()`.
- Notes: Respects speed limiter (heartbeat or net time); blocks until enough time has elapsed. Prediction phase runs speculatively without speed limit.

### update_world_elements_one_tick
- Signature: `static int update_world_elements_one_tick(void)`
- Purpose: Advance all entities by one game tick; return completion status.
- Inputs: None (reads GameQueue).
- Outputs/Return: `kUpdateNormalCompletion`, `kUpdateGameOver`, or `kUpdateChangeLevel`.
- Side effects: Updates lights, media, platforms, control panels, players (via GameQueue), projectiles, monsters, effects, scenery, items, animated textures, chase camera, network game state; increments tick_count.
- Calls: `L_Call_Idle()`, `update_lights()`, `update_medias()`, `update_platforms()`, `update_control_panels()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()`, `recreate_objects()`, `handle_random_sound_image()`, `animate_scenery()`, `animate_items()`, `AnimTxtr_Update()`, `ChaseCam_Update()`, `update_net_game()`, `check_level_change()`, `game_is_over()`, `L_Call_PostIdle()`.
- Notes: Assumes exactly 1 tick (no delta-time); order of updates matters for correctness.

### entering_map
- Signature: `bool entering_map(bool restoring_saved)`
- Purpose: Load and initialize a new level; called after map geometry is loaded but before first update.
- Inputs: `restoring_saved` ΓÇö whether a save game is being restored (skips Lua init if true).
- Outputs/Return: `true` if successful; `false` triggers `leaving_map()` cleanup.
- Side effects: Initializes monsters/paths, marks and loads shape collections, loads sounds, runs Lua script, syncs network, checks weapon legality, resets action queues, clears fades, sets up chat callbacks.
- Calls: `initialize_monsters_for_new_level()`, `reset_paths()`, `mark_environment_collections()`, `RunLuaScript()`, `MarkLuaCollections()`, `load_collections()`, `load_all_monster_sounds()`, `load_all_game_sounds()`, `NetSync()`, `check_player_weapons_for_environment_change()`, `initialize_net_game()`, `randomize_scenery_shapes()`, `reset_action_queues()`, `L_Call_Init()`, `NetSetChatCallbacks()`, `stop_fade()`, `set_fade_effect()`, `leaving_map()`.
- Notes: Restoring a save game skips Pfhortran (Lua) script initialization. Order is critical for consistency.

### leaving_map
- Signature: `void leaving_map(void)`
- Purpose: Cleanup when departing a level; remove transient entities and unload resources.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Removes all projectiles and non-persistent effects, marks collections for unload, closes Lua script, deactivates console input, stops music and all sounds, unloads custom sounds.
- Calls: `remove_all_projectiles()`, `remove_all_nonpersistent_effects()`, `mark_environment_collections()`, `mark_all_monster_collections()`, `mark_player_collections()`, `mark_map_collections()`, `MarkLuaCollections()`, `L_Call_Cleanup()`, `CloseLuaScript()`, `NetSetChatCallbacks()`, `Console::instance()->deactivate_input()`, `Music::instance()->StopLevelMusic()`, `SoundManager::instance()->StopAllSounds()`, `SoundManager::instance()->UnloadCustomSounds()`.
- Notes: Collections are marked but not immediately freed; they are freed when `load_collections()` is called on the next level.

### changed_polygon
- Signature: `void changed_polygon(short original_polygon_index, short new_polygon_index, short player_index)`
- Purpose: Handle polygon entry/exit triggers (monster activation, platform control, damage zones, light switches, item triggers).
- Inputs: `original_polygon_index` (NONE if unknown), `new_polygon_index`, `player_index` (NONE if non-player entity).
- Outputs/Return: None.
- Side effects: May activate monsters, toggle lights, trigger platforms, set player status flags (must-explore).
- Calls: `get_polygon_data()`, `get_player_data()`, `activate_nearby_monsters()`, `trigger_nearby_items()`, `set_light_status()`, `platform_was_entered()`, `try_and_change_platform_state()`.
- Notes: Checks polygon type and permutation to determine action; only players trigger certain polygon types.

### calculate_level_completion_state
- Signature: `short calculate_level_completion_state(void)`
- Purpose: Evaluate whether mission objectives are met and return completion status.
- Inputs: None (reads static_world and dynamic_world mission flags).
- Outputs/Return: `_level_finished`, `_level_unfinished`, or `_level_failed`.
- Side effects: None.
- Calls: `live_aliens_on_map()`, `unretrieved_items_on_map()`, `untoggled_repair_switches_on_level()`.
- Notes: Failure condition: rescue mission with >50% civilian casualties. Completion requires all mission types' objectives satisfied.

### calculate_damage
- Signature: `short calculate_damage(struct damage_definition *damage)`
- Purpose: Compute final damage amount from a definition, applying random variance and difficulty scaling.
- Inputs: `damage` struct with type, base, random, scale, flags.
- Outputs/Return: Final damage value.
- Side effects: None (pure computation).
- Calls: `global_random()`.
- Notes: Alien damage reduced at lower difficulty levels (wuss: 50%, easy: 25%). Fixed-point math via `FIXED_INTEGERAL_PART()`.

### enter_predictive_mode / exit_predictive_mode
- Signature: `static void enter_predictive_mode(void)` / `static void exit_predictive_mode(void)`
- Purpose: Save/restore player and monster state for client-side movement prediction.
- Inputs: None (modifies static saved-state arrays).
- Outputs/Return: None.
- Side effects: Copies player, monster, object data to/from static arrays; defers polygon object list manipulations on exit.
- Calls: `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_random_seed()`, `remove_object_from_polygon_object_list()`, `deferred_add_object_to_polygon_object_list()`, `perform_deferred_polygon_object_list_manipulations()`.
- Notes: `sPredictedTicks == 0` indicates not in prediction mode. Interface flags are preserved on restore to allow inventory scrolling. Sanity checks log warnings if tick_count or random seed diverges.

### overlay_queue_with_queue_into_queue
- Signature: `static bool overlay_queue_with_queue_into_queue(ActionQueues* inBaseQueues, ActionQueues* inOverlayQueues, ActionQueues* inOutputQueues)`
- Purpose: Dequeue one tick of action flags per player from base queue, override with overlay queue if present, enqueue into output.
- Inputs: Base queue (primary source), overlay queue (override source, may be NULL), output queue (destination).
- Outputs/Return: `true` if all players had flags; `false` if any player's base queue empty.
- Side effects: Modifies all three queues.
- Calls: `countActionFlags()`, `dequeueActionFlags()`, `enqueueActionFlags()`.
- Notes: Always dequeues from base even if overridden; ensures base queue advances. Used to merge Lua/real input.

## Control Flow Notes
This file is central to the per-frame game update cycle. After input/network message processing, `update_world()` is called each frame:
1. Fetches action flags from queues (real input, Lua, network).
2. If enough game time has elapsed (speed limiter), advances game logic one tick via `update_world_elements_one_tick()`.
3. Updates UI/fades if any ticks elapsed.
4. If prediction is enabled, speculatively simulates additional ticks using unconfirmed client input, then restores actual state.
5. Returns whether redraw is needed.

Level loading/unloading is orchestrated by `entering_map()` (on level start) and `leaving_map()` (on departure).

## External Dependencies
- **Game subsystems** (defined elsewhere): `map.h`, `render.h`, `interface.h`, `flood_map.h`, `effects.h`, `monsters.h`, `projectiles.h`, `player.h`, `network.h`, `scenery.h`, `platforms.h`, `lightsource.h`, `media.h`, `items.h`, `weapons.h`.
- **Managers**: `Music.h`, `SoundManager.h`, `game_window.h`, `network_games.h`, `tags.h`, `AnimatedTextures.h`, `ChaseCam.h`, `OGL_Setup.h`.
- **Scripting & I/O**: `lua_script.h`, `ActionQueues.h`, `Logging.h`, `Console.h`, `screen.h`, `shell.h`.
- **Utilities**: `cseries.h` (standard types and macros).

**Key external symbols** (defined elsewhere):
- `dynamic_world`, `static_world` ΓÇö global game state pointers.
- `GetRealActionQueues()`, `GetLuaActionQueues()` ΓÇö input source accessors.
- `map_polygons[]` ΓÇö polygon array (from map.h).
- Entity data accessors: `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_polygon_data()`.
- Subsystem update functions: `update_players()`, `move_monsters()`, `update_effects()`, etc.
- Network functions: `NetProcessMessagesInGame()`, `NetSync()`, `game_is_networked`, `NetGetNetTime()`.
