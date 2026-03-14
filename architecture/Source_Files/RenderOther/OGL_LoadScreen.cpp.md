# Source_Files/RenderOther/OGL_LoadScreen.cpp

## File Purpose
Implements OpenGL-based loading screens for the Aleph One game engine. Handles image file loading, aspect-ratio-aware scaling, and rendering with an optional progress bar overlay during asset loading.

## Core Responsibilities
- Singleton instance management via lazy initialization
- Image file loading and SDL surface conversion with endianness handling
- Aspect-ratio-aware or stretch-fill image scaling and centering
- Progress bar rendering (vertical or horizontal) with configurable dimensions and colors
- OpenGL blitter initialization, setup, and cleanup
- Resource deallocation and state reset

## Key Types / Data Structures
| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|---|---|
| OGL_LoadScreen | class | Singleton managing load screen display state and rendering |
| ImageDescriptor | class (external) | Stores image pixel data and metadata |
| OGL_Blitter | class (external) | OpenGL renderer for textured quads |
| SDL_Surface | struct (external) | SDL pixel surface representation |
| SDL_Rect | struct (external) | Screen-space rectangle (position + size) |
| rgb_color | struct (external) | RGB color triplet |

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|---|---|
| instance_ | OGL_LoadScreen* | static member | Singleton instance pointer |

## Key Functions / Methods

### instance()
- Signature: `static OGL_LoadScreen *instance()`
- Purpose: Return singleton instance, creating it lazily if needed
- Inputs: None
- Outputs/Return: Pointer to the static OGL_LoadScreen instance
- Side effects: Heap allocation on first call; not thread-safe
- Calls: OGL_LoadScreen constructor
- Notes: Standard lazy-init singleton pattern

### Start()
- Signature: `bool Start()`
- Purpose: Load image file, compute aspect-ratio-preserving scaling, initialize blitter, and prepare first render
- Inputs: Member variables (path, stretch, x, y, w, h; set via Set())
- Outputs/Return: true on success; false if path empty or image load fails
- Side effects: File I/O; SDL surface creation; OGL_Blitter heap allocation (old one deleted); screen clear; calls Progress(0)
- Calls: File.SetNameWithPath(), image.LoadFromFile(), SDL_CreateRGBSurfaceFrom(), bound_screen(), OGL_ClearScreen(), Progress()
- Notes: Computes scaled dimensions respecting aspect ratio unless stretch=true; handles endian-specific RGBA channel ordering; stores SDL_Surface pointer in blitter

### Stop()
- Signature: `void Stop()`
- Purpose: Finalize rendering and transition away from load screen
- Inputs: None
- Outputs/Return: None
- Side effects: Clears screen, swaps buffers, calls Clear()
- Calls: OGL_ClearScreen(), OGL_SwapBuffers(), Clear()
- Notes: Simple wrapper; actual cleanup deferred to Clear()

### Progress(const int progress)
- Signature: `void Progress(const int progress)`
- Purpose: Render load screen image and animated progress bar at given completion percentage
- Inputs: progress value 0ΓÇô100 (percentage complete)
- Outputs/Return: None
- Side effects: OpenGL state changes (clear, matrix setup, texture binding, color, vertex submission); buffer swap
- Calls: OGL_ClearScreen(), blitter->SetupMatrix(), blitter->Draw(), glBindTexture(), glColor3us() (2├ù), glBegin(), glVertex3f() (8├ù), glEnd() (2├ù), blitter->RestoreMatrix(), OGL_SwapBuffers()
- Notes: Draws background quad in colors[0], then foreground progress bar in colors[1]; bar clips height (vertical) or width (horizontal) based on dimensions; uses screen coordinates (x, y, w, h)

### Set(const vector<char> Path, bool Stretch)
- Signature: `void Set(const vector<char> Path, bool Stretch)`
- Purpose: Configure load screen without progress bar
- Inputs: File path (char vector), stretch flag
- Outputs/Return: None
- Side effects: Sets member variables; clears image; sets useProgress=false
- Calls: Set(Path, Stretch, 0, 0, 0, 0)
- Notes: Convenience overload; delegates to 6-parameter version with zero progress bar dimensions

### Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)
- Signature: `void Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)`
- Purpose: Configure load screen with progress bar position, size, and enable rendering
- Inputs: File path, stretch flag, progress bar position (X, Y) and dimensions (W, H)
- Outputs/Return: None
- Side effects: Sets all member variables; clears image; enables progress bar
- Calls: image.Clear()
- Notes: Stores configuration; actual image load deferred to Start(); resets percent to 0

### Clear()
- Signature: `void Clear()`
- Purpose: Release all resources and reset to uninitialized state
- Inputs: None
- Outputs/Return: None
- Side effects: Screen clear (2├ù); buffer swap; blitter delete; image and path cleared; use=false, useProgress=false
- Calls: OGL_ClearScreen() (├ù2), OGL_SwapBuffers(), image.Clear(), delete blitter
- Notes: Sets blitter to NULL after delete; double screen clear is defensive/redundant

## Control Flow Notes
Intended usage flow: **Set() ΓåÆ Start() ΓåÆ [Progress(%) loop] ΓåÆ Stop() ΓåÆ Clear()**

- **Init phase**: Set() stores configuration (path, scaling, progress bar bounds)
- **Load phase**: Start() loads image and initializes blitter; Progress(0) renders first frame
- **Progress phase**: External code calls Progress(percent) in a load loop, rendering updated progress bar
- **Teardown**: Stop() finalizes current frame; Clear() deallocates all resources

## External Dependencies
- **Headers**: OGL_LoadScreen.h, screen.h, OGL_Blitter.h, ImageLoader.h, cseries.h
- **OpenGL**: gl.h / GL/gl.h ΓÇö glBindTexture, glColor3us, glBegin, glVertex3f, glEnd
- **SDL**: SDL_CreateRGBSurfaceFrom, SDL_Rect, SDL_Surface, SDL_GetVideoSurface
- **Game engine**: OGL_ClearScreen(), OGL_SwapBuffers(), bound_screen() (defined elsewhere); FileSpecifier, ImageDescriptor, OGL_Blitter classes
- **Platform**: WindowPtr, GetWindowPort, GetPortBounds (Mac OS Classic only, guarded by #if defined(mac))
