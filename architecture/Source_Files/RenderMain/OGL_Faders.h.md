# Source_Files/RenderMain/OGL_Faders.h

## File Purpose
Declares the OpenGL renderer's fader system interface, which manages fade-in/fade-out screen transitions with color and transparency. Part of Aleph One's rendering pipeline to handle visual fading effects.

## Core Responsibilities
- Define the `OGL_Fader` data structure for storing fade parameters
- Declare query functions to check fader state and access fader queue entries
- Declare the main fader rendering function that applies fades to a rectangular viewport region
- Define fader queue type categories (liquid vs. other)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Fader` | struct | Encapsulates fade effect data: fade type, and four-channel color (RGBA) |
| FaderQueue enum | anonymous enum | Categorizes fader queue entries (liquid, other, plus count) |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_FaderActive
- Signature: `bool OGL_FaderActive()`
- Purpose: Query whether OpenGL-rendering faders are currently in use
- Inputs: None
- Outputs/Return: Boolean indicating active fader state
- Side effects: None (query only)
- Calls: (not visible)
- Notes: Used to determine if fading effects should be processed in the render pipeline

### GetOGL_FaderQueueEntry
- Signature: `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- Purpose: Retrieve a fader entry from the fader queue by index
- Inputs: `Index` ΓÇô queue position (context suggests 0 or 1 for Liquid/Other)
- Outputs/Return: Pointer to `OGL_Fader` struct
- Side effects: None (accessor)
- Calls: (not visible)
- Notes: Provides read/write access to queued fader data

### OGL_DoFades
- Signature: `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- Purpose: Render active faders to the screen within the specified rectangular region
- Inputs: `Left, Top, Right, Bottom` ΓÇô viewport bounds
- Outputs/Return: Boolean indicating whether faders were actually rendered
- Side effects: Modifies OpenGL state (color, blending, geometry rendering)
- Calls: (not visible; calls OpenGL rendering functions)
- Notes: Entry point for the frame's fader pass; only executes if faders are active

## Control Flow Notes
This module is called during the render pipeline's post-scene pass to apply fade overlays. `OGL_DoFades()` is the frame-level entry point; it checks `OGL_FaderActive()` and accesses the fader queue via `GetOGL_FaderQueueEntry()` to draw accumulated fades.

## External Dependencies
- OpenGL (implicit; referenced in file comments)
- Fader queue management (implementation elsewhere; accessed via `GetOGL_FaderQueueEntry()`)
- Type definition for `NONE` constant (defined elsewhere)
