# Source_Files/Misc/preference_dialogs.h

## File Purpose
Defines an abstract base class for OpenGL graphics preferences dialogs in the Aleph One game engine. Uses the abstract factory pattern to create platform-specific dialog implementations (SDL or Carbon) at link-time. Manages UI widgets for configuring OpenGL rendering parameters.

## Core Responsibilities
- Abstract factory for creating platform-specific preference dialogs
- UI widget management for OpenGL graphics settings (resolution, filtering, effects, etc.)
- Dialog lifecycle control (show, run, and dismiss preferences)
- Data binding between UI widgets and OpenGL configuration options
- Organization of graphics preferences into logical widget groups (toggles, selectors, color pickers)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OpenGLDialog | class (abstract) | Base class for OpenGL preferences dialog; defines interface and shared widget layout |

## Global / File-Static State
None.

## Key Functions / Methods

### Create
- Signature: `static std::auto_ptr<OpenGLDialog> Create()`
- Purpose: Abstract factory method; returns concrete platform-specific implementation determined at link-time
- Inputs: None
- Outputs/Return: `std::auto_ptr<OpenGLDialog>` pointing to derived platform-specific instance
- Side effects: Allocates heap memory for dialog instance
- Calls: Not visible in this file (implemented elsewhere)
- Notes: Enables single-source codebase with platform-specific dialog implementations

### OpenGLPrefsByRunning
- Signature: `void OpenGLPrefsByRunning()`
- Purpose: Public entry point to display and run the preferences dialog
- Inputs: None
- Outputs/Return: void
- Side effects: Likely calls `Run()` and `Stop()` to manage dialog lifecycle
- Calls: Presumably calls pure virtual `Run()` internally
- Notes: Main user-facing API for launching the dialog

### Run (pure virtual)
- Signature: `virtual bool Run() = 0`
- Purpose: Abstract method for platform-specific dialog execution and event loop
- Inputs: None
- Outputs/Return: `bool` (likely indicates user acceptance/cancellation)
- Side effects: Not inferable from this file; platform-specific
- Calls: N/A (pure virtual)
- Notes: Implemented in derived classes (SDL/Carbon variants)

### Stop (pure virtual)
- Signature: `virtual void Stop(bool result) = 0`
- Purpose: Abstract method for platform-specific dialog termination
- Inputs: `bool result` ΓÇô likely indicates whether changes were accepted
- Outputs/Return: void
- Side effects: Not inferable from this file; platform-specific
- Calls: N/A (pure virtual)
- Notes: Implemented in derived classes

## Control Flow Notes
This class fits into the **preferences initialization / runtime configuration phase**:
1. User launches graphics preferences dialog
2. `OpenGLPrefsByRunning()` is called
3. Platform-specific `Run()` creates UI, displays dialog, and waits for user input
4. User clicks OK/Cancel ΓåÆ `Stop(result)` is invoked
5. Dialog is destroyed; preferences applied to engine

## External Dependencies
- **shared_widgets.h**: `ButtonWidget`, `ToggleWidget`, `SelectorWidget`, `SelectSelectorWidget`, `ColourPickerWidget` ΓÇô abstract UI widget classes (implementations selected at compile-time via SDL or Carbon conditional include)
- **OGL_Setup.h**: 
  - `OGL_NUMBER_OF_TEXTURE_TYPES` constant (enum value defining array sizes)
  - Defines OpenGL configuration structures referenced by this dialog
