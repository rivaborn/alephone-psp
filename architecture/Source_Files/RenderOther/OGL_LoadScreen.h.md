# Source_Files/RenderOther/OGL_LoadScreen.h

## File Purpose
Singleton class managing OpenGL-rendered load screens displayed during game level loading. Displays a background image with optional progress indicator percentage. Coordinates image loading, texture management, and on-screen rendering via OpenGL blitting.

## Core Responsibilities
- Load and cache image files for display during level transitions
- Render background image via OpenGL with optional stretching/positioning
- Display and update progress percentage indicator during loading
- Manage OpenGL texture references and lifecycle
- Provide start/stop lifecycle controls for load screen visibility
- Store and expose UI color palette for progress indicator rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_LoadScreen` | class | Singleton managing load screen rendering |
| `ImageDescriptor` | class (from ImageLoader.h) | Loaded image data and metadata |
| `OGL_Blitter` | class (from OGL_Blitter.h) | Renders textured quads via OpenGL |
| `rgb_color` | struct (from cseries.h) | RGB color triplet |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance_` | `OGL_LoadScreen*` | static | Singleton instance pointer |

## Key Functions / Methods

### instance()
- **Signature:** `static OGL_LoadScreen *instance()`
- **Purpose:** Return singleton instance; create if needed
- **Inputs:** None
- **Outputs/Return:** Pointer to global OGL_LoadScreen instance
- **Side effects:** Creates singleton on first call
- **Calls:** Not inferable from header
- **Notes:** Classic singleton pattern; thread-safety not inferable

### Start()
- **Signature:** `bool Start()`
- **Purpose:** Initialize and display load screen; prepare OpenGL context
- **Inputs:** None (uses configured image path and dimensions)
- **Outputs/Return:** `true` if successful, `false` on error
- **Side effects:** Allocates `blitter`, configures OpenGL state, sets `use=true`
- **Calls:** Not inferable from header
- **Notes:** Must be paired with `Stop()`; called before level load begins

### Stop()
- **Signature:** `void Stop()`
- **Purpose:** Disable load screen and clean up OpenGL resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes `blitter`, releases texture, sets `use=false`
- **Calls:** Not inferable from header

### Progress(int percent)
- **Signature:** `void Progress(const int percent)`
- **Purpose:** Update progress indicator; typically called per-frame during loading
- **Inputs:** `percent` ΓÇô progress value 0ΓÇô100
- **Outputs/Return:** None
- **Side effects:** Stores `percent`, triggers redraw (likely)
- **Calls:** Not inferable from header
- **Notes:** `useProgress` must be `true` for indicator to display

### Set() overloads
- **Signature:** 
  - `void Set(const vector<char> Path, bool Stretch)`
  - `void Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)`
- **Purpose:** Load image and configure display (stretch, position/size)
- **Inputs:** 
  - `Path` ΓÇô file path as char vector
  - `Stretch` ΓÇô scale to fit viewport
  - `X, Y, W, H` ΓÇô optional destination rectangle on screen
- **Outputs/Return:** None
- **Side effects:** Loads `image`, updates `path`, sets position/stretch flags
- **Calls:** ImageLoader functions (not visible)
- **Notes:** First form uses full viewport; second uses specified rect

### Clear()
- **Signature:** `void Clear()`
- **Purpose:** Unload current image and reset configuration
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `image`, resets `path`, `x`, `y`, `w`, `h`

### Use(), Colors()
- **Signature:** `bool Use()`, `rgb_color *Colors()`
- **Purpose:** Query load screen enabled state; access color palette
- **Inputs:** None
- **Outputs/Return:** `use` flag; pointer to 2-element `rgb_color` array
- **Side effects:** None
- **Notes:** `Colors()` likely used for rendering progress bar foreground/background

## Control Flow Notes
Load screen lifecycle integrates into engine's level-load sequence:
1. **Start** ΓÇô called before unloading current level; sets up OpenGL context and image
2. **Progress loop** ΓÇô per-frame during loading (I/O, asset decompression, etc.), update progress and redraw
3. **Stop** ΓÇô called when level load complete; cleans up and hides overlay

Not inferable from this header whether rendering is push-driven (explicit draw call) or pull-driven (automatic per `Progress()` call).

## External Dependencies
- **OpenGL:** Platform-specific headers (`OpenGL/gl.h`, `GL/gl.h`, etc.)
- **OGL_Blitter** (Source_Files/RenderOther/OGL_Blitter.h) ΓÇô textured quad rendering
- **ImageLoader** (Source_Files/RenderMain/ImageLoader.h) ΓÇô image file loading and `ImageDescriptor`
- **cseries.h** ΓÇô cross-platform types (`vector`, SDL, `rgb_color`)
- **SDL** ΓÇô low-level graphics/windowing (transitively via cseries.h)
