# Source_Files/RenderOther/screen_definitions.h

## File Purpose
Defines base resource IDs for all screen types in the rendering system. These constants serve as anchors for a three-tiered numbering scheme: 8-bit variants use the base ID, 16-bit variants add +10,000, and 32-bit variants add +20,000.

## Core Responsibilities
- Provide compile-time constants for screen resource identification
- Establish a consistent offset pattern (100-point gaps) for different screen categories
- Support multi-bit-depth asset management without hardcoding variant IDs

## Key Types / Data Structures
None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| INTRO_SCREEN_BASE | enum constant (int) | global | Base ID for intro sequence screens (value: 1000) |
| MAIN_MENU_BASE | enum constant (int) | global | Base ID for main menu screens (value: 1100) |
| PROLOGUE_SCREEN_BASE | enum constant (int) | global | Base ID for prologue narrative screens (value: 1200) |
| EPILOGUE_SCREEN_BASE | enum constant (int) | global | Base ID for epilogue narrative screens (value: 1300) |
| CREDIT_SCREEN_BASE | enum constant (int) | global | Base ID for credits roll screens (value: 1400) |
| CHAPTER_SCREEN_BASE | enum constant (int) | global | Base ID for chapter introduction screens (value: 1500) |
| COMPUTER_INTERFACE_BASE | enum constant (int) | global | Base ID for in-game computer interface screens (value: 1600) |
| INTERFACE_PANEL_BASE | enum constant (int) | global | Base ID for HUD/UI panel screens (value: 1700) |
| FINAL_SCREEN_BASE | enum constant (int) | global | Base ID for end-game final screens (value: 1800) |

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This file is purely declarativeΓÇöconsumed by resource loaders and screen managers at runtime to locate image assets by calculated ID.

## External Dependencies
None beyond C standard library (enum syntax only).
