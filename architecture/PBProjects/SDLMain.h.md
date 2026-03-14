# PBProjects/SDLMain.h

## File Purpose
Defines the Objective-C interface for the main application controller in a Cocoa-based SDL game application. Declares menu action handlers for game lifecycle operations (new, open, save) and preferences/help, functioning as the primary entry point for macOS GUI integration.

## Core Responsibilities
- Application controller interface for Cocoa framework integration
- Menu action handler declarations for game lifecycle (new game, open/save game)
- Preferences menu action handler
- Help action handler
- Acts as the delegate/responder for user-initiated menu events

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDLMain | Interface (class) | Main application controller, inherits from NSObject |

## Global / File-Static State
None.

## Key Methods
All methods are IBAction handlers (callback entry points from Interface Builder / macOS menu system):

- `prefsMenu:` ΓÇô Preferences menu action
- `newGame:` ΓÇô New game menu action
- `openGame:` ΓÇô Open game menu action
- `saveGame:` ΓÇô Save current game menu action
- `saveGameAs:` ΓÇô Save game with alternate path/name menu action
- `help:` ΓÇô Help menu action

Each signature: `- (IBAction)<methodName>:(id)sender;` where `sender` is the UI control that triggered the action.

## Control Flow Notes
Sits at the macOS Cocoa application entry point. Methods are invoked by the Cocoa event loop when user interacts with corresponding menu items. Actual implementation is in SDLMain.m (not shown). Likely calls into the SDL game engine after handling Cocoa-specific concerns.

## External Dependencies
- `#import <Cocoa/Cocoa.h>` ΓÇô macOS Cocoa framework (provides NSObject, UIKit integration)
- `NSObject` ΓÇô Base class from Foundation framework (part of Cocoa)
