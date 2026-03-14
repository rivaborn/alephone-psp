# Source_Files/Input/ISp_Support.h

## File Purpose
Header declaring the public interface for InputSprocket support in the Marathon/Aleph One game engine. InputSprocket (ISp) is a legacy macOS input management API. This file provides functions to initialize, configure, and manage input devices during gameplay.

## Core Responsibilities
- Initialize and shut down the InputSprocket system
- Start/stop input event processing during gameplay sessions
- Query active input device type (keyboard vs. other devices)
- Enumerate and test available input devices
- Configure game-specific input control mappings for InputSprocket

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_ISp
- Signature: `void initialize_ISp(void)`
- Purpose: Initialize and set up the InputSprocket system during engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes InputSprocket resources and state

### ShutDown_ISp
- Signature: `void ShutDown_ISp(void)`
- Purpose: Clean up and shut down the InputSprocket system during engine shutdown
- Inputs: None
- Outputs/Return: None
- Side effects: Frees InputSprocket resources

### Start_ISp
- Signature: `void Start_ISp(void)`
- Purpose: Begin processing input events during a game session
- Inputs: None
- Outputs/Return: None
- Side effects: Activates input polling/event handling

### Stop_ISp
- Signature: `void Stop_ISp(void)`
- Purpose: Pause input event processing
- Inputs: None
- Outputs/Return: None
- Side effects: Suspends input polling

### ISp_IsUsingKeyboard
- Signature: `bool ISp_IsUsingKeyboard(void)`
- Purpose: Query whether the active input device is a keyboard
- Inputs: None
- Outputs/Return: Boolean indicating keyboard usage
- Side effects: None (read-only query)

### InputSprocketTestElements
- Signature: `long InputSprocketTestElements(void)`
- Purpose: Test and enumerate available input devices/elements
- Inputs: None
- Outputs/Return: Long integer (likely device count or status code)
- Side effects: May probe hardware state

### ConfigureMarathonISpControls
- Signature: `void ConfigureMarathonISpControls(void)`
- Purpose: Configure game-specific control mappings for InputSprocket
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies InputSprocket configuration for Marathon controls

## Control Flow Notes
Fits into the engine lifecycle: `initialize_ISp()` during startup, `Start_ISp()` when a game session begins, `Stop_ISp()` when pausing/exiting gameplay, `ShutDown_ISp()` during engine shutdown. Input device queries integrate into the per-frame input polling pipeline.

## External Dependencies
- InputSprocket API (legacy macOS input framework) ΓÇö defined elsewhere
