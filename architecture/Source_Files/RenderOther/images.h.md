# Source_Files/RenderOther/images.h

## File Purpose

Image and resource management subsystem for the Aleph One game engine. Provides interfaces to load and manipulate game artwork (pictures, sounds, text) from both game resources and scenario files, with dual-platform support for MacOS resource forks and SDL-based systems.

## Core Responsibilities

- Initialize and manage the image/resource loading system
- Load and check existence of picture resources from game files and scenario files
- Calculate and manage color lookup tables (CLUTs) for image rendering
- Draw full-screen pictures and handle picture scrolling with text overlays
- Load sound and text resources from scenario files
- Convert MacOS PICT resources to SDL surfaces
- Perform surface transformations (rescaling, tiling) on SDL graphics objects
- Abstract away platform-specific resource fork handling via FileSpecifier

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LoadedResource` | class (from FileHandler.h) | Wrapper managing lifetime of loaded resource data (auto-unload on destruction) |
| `FileSpecifier` | class (from FileHandler.h) | Abstract file path/specification; encapsulates MacOS FSSpec or SDL path |
| `color_table` | struct (defined elsewhere) | Color palette/lookup table for 8-bit rendering |
| `SDL_Surface` | typedef (from SDL) | Pixel buffer abstraction for graphics rendering |
| CLUT source enum | enum constants | `CLUTSource_Images`, `CLUTSource_Scenario` ΓÇö selects which file is source of color table |

## Global / File-Static State

None.

## Key Functions / Methods

### initialize_images_manager
- **Signature:** `extern void initialize_images_manager(void);`
- **Purpose:** Initialize the image manager subsystem at engine startup.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Initializes global state for resource loading.
- **Calls:** Not inferable from this file.
- **Notes:** Called once during engine initialization.

### images_picture_exists / scenario_picture_exists
- **Signature:** `extern bool images_picture_exists(int base_resource);` (and scenario variant)
- **Purpose:** Check if a picture resource exists without loading it.
- **Inputs:** `base_resource` ΓÇö resource ID to check.
- **Outputs/Return:** Boolean; true if resource exists.
- **Side effects:** None (read-only check).
- **Calls:** Not inferable.

### calculate_picture_clut
- **Signature:** `extern struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number);`
- **Purpose:** Compute a color lookup table from a picture resource.
- **Inputs:** `CLUTSource` (Images or Scenario), `pict_resource_number` (resource ID).
- **Outputs/Return:** Pointer to allocated color_table struct.
- **Side effects:** Allocates memory; caller responsible for deallocation.
- **Calls:** Not inferable.
- **Notes:** Source selector allows pulling CLUT from different resource files.

### set_scenario_images_file
- **Signature:** `extern void set_scenario_images_file(FileSpecifier& File);`
- **Purpose:** Designate which scenario file is the active source for scenario resources.
- **Inputs:** Reference to FileSpecifier identifying the scenario file.
- **Outputs/Return:** None.
- **Side effects:** Updates global resource file state.
- **Calls:** Not inferable.

### draw_full_screen_pict_resource_from_images / draw_full_screen_pict_resource_from_scenario
- **Signature:** `extern void draw_full_screen_pict_resource_from_images(int pict_resource_number);` (and scenario variant)
- **Purpose:** Load and render a full-screen picture resource directly to the frame buffer.
- **Inputs:** `pict_resource_number` (resource ID).
- **Outputs/Return:** None.
- **Side effects:** Modifies frame buffer / screen display.
- **Calls:** Not inferable.
- **Notes:** Likely used for splash screens, menus, cutscenes.

### scroll_full_screen_pict_resource_from_scenario
- **Signature:** `extern void scroll_full_screen_pict_resource_from_scenario(int pict_resource_number, bool text_block);`
- **Purpose:** Draw a scrolling full-screen picture, optionally with a text block overlay.
- **Inputs:** `pict_resource_number` (resource ID), `text_block` (whether to render text).
- **Outputs/Return:** None.
- **Side effects:** Modifies frame buffer; may animate/scroll over multiple frames.
- **Calls:** Not inferable.

### get_picture_resource_from_images / get_picture_resource_from_scenario
- **Signature:** `extern bool get_picture_resource_from_images(int base_resource, LoadedResource& PictRsrc);` (and scenario variant)
- **Purpose:** Load a picture resource into a LoadedResource wrapper.
- **Inputs:** `base_resource` (resource ID), reference to LoadedResource to populate.
- **Outputs/Return:** Boolean; true if successfully loaded.
- **Side effects:** Loads resource data into LoadedResource; memory owned by LoadedResource.
- **Calls:** Not inferable.
- **Notes:** Caller does not manage resource memory (RAII via LoadedResource).

### get_sound_resource_from_scenario / get_text_resource_from_scenario
- **Signature:** `extern bool get_sound_resource_from_scenario(int resource_number, LoadedResource& SoundRsrc);` (and text variant)
- **Purpose:** Load sound or text resource from scenario file.
- **Inputs:** `resource_number` (resource ID), reference to LoadedResource.
- **Outputs/Return:** Boolean; true if loaded.
- **Side effects:** Loads resource data.
- **Calls:** Not inferable.

### picture_to_surface
- **Signature:** `extern SDL_Surface *picture_to_surface(LoadedResource &rsrc);` (SDL-only)
- **Purpose:** Convert a MacOS PICT resource to an SDL_Surface for rendering.
- **Inputs:** Reference to LoadedResource containing PICT data.
- **Outputs/Return:** Allocated SDL_Surface pointer; caller owns.
- **Side effects:** Allocates SDL_Surface memory.
- **Calls:** Not inferable.
- **Notes:** Conditional on SDL compilation; bridges MacOS resource format to SDL graphics.

### rescale_surface / tile_surface
- **Signature:** `extern SDL_Surface *rescale_surface(SDL_Surface *s, int width, int height);` / `extern SDL_Surface *tile_surface(SDL_Surface *s, int width, int height);` (SDL-only)
- **Purpose:** Transform an SDL surface by scaling or tiling to a target dimension.
- **Inputs:** Source surface, target width, height.
- **Outputs/Return:** New SDL_Surface; caller owns.
- **Side effects:** Allocates new surface.
- **Calls:** Not inferable.

## Control Flow Notes

This header is part of the **render pipeline**, likely invoked during:

- **Initialization:** `initialize_images_manager()` called at engine startup.
- **Frame/Update:** Resource existence checks (`images_picture_exists`) to conditionally load/render assets.
- **Render:** Drawing functions (`draw_full_screen_pict_resource_*`) called when UI or cutscenes require display.
- **Cleanup:** LoadedResource destructors auto-unload when done.

The dual MacOS/SDL support suggests this engine abstractively bridges platform-specific resource systems (MacOS resource forks) to portable SDL graphics.

## External Dependencies

- **FileHandler.h** ΓÇö `LoadedResource`, `FileSpecifier` abstractions for file I/O and resource lifetime.
- **SDL.h** (conditional) ΓÇö `SDL_Surface`, `SDL_RWops` for graphics and file I/O on non-MacOS platforms.
- **Implicit MacOS APIs** ΓÇö On `mac` platform: Carbon/Classic resource manager (`GetCTable`, etc.) via FileHandler.h.
