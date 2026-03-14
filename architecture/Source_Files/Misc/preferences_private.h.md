# Source_Files/Misc/preferences_private.h

## File Purpose
Internal constants and enumerations for the preferences UI system. Defines dialog item identifiers (ditl), OSType signatures for NIB resources, and control tags for each preferences pane (graphics, player, sound, input, environment). Original content ripped from preferences_macintosh.cpp.

## Core Responsibilities
- Define dialog item identifier enums for each preference pane (ditlGRAPHICS, ditlPLAYER, ditlSOUND, ditlINPUT, ditlENVIRONMENT)
- Provide OSType signatures for marking panes and their control categories
- Supply control index constants (e.g., iCHOOSE_MONITOR, iDIFFICULTY_LEVEL, iMOUSE_CONTROL)
- Enumerate preference groups and their counts
- Support Mac NIB (Interface Builder) file integration via Core Foundation types

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Window_Preferences | CFStringRef | global (conditional) | NIB window resource identifier |
| iTABS | int | global (conditional) | Number of preference panes (hardcoded to 3) |
| Sig_Pane, Sig_Graphics, Sig_Sound, Sig_Player, Sig_Input, Sig_Environment | OSType | global (conditional) | Mac resource type signatures for pane containers and control groups |

## Key Functions / Methods
NoneΓÇöthis is a constants-only header.

## Control Flow Notes
Not inferable. This file is a passive declarations file. Constants are consumed by preference UI implementers to identify and manage dialog controls, but the file itself has no control flow.

## External Dependencies
- `CFStringRef`, `CFSTR` ΓÇô Core Foundation (conditionally included via `USES_NIBS`)
- `OSType` ΓÇô Mac classic four-character code type
- All definitions are conditional on `#ifdef USES_NIBS` (except enums)

## Notes
- Comments attribute additions to "LP" (likely Loren Petrich) and Ian Rickard, suggesting iterative development by multiple contributors.
- Hardcoded enum values (4001ΓÇô4005) correspond to Mac dialog template resource IDs.
- The PREFERENCES_GROUPS enum mirrors the pane structure, supporting 5 groups with a terminator count.
