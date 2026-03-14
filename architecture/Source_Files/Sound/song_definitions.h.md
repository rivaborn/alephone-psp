# Source_Files/Sound/song_definitions.h

## File Purpose
Defines C structures and constants for organizing game music tracks into segments. Part of the Aleph One engine's audio subsystem. Provides a data-driven format for specifying song metadata including introduction, chorus, and trailer segments with looping behavior.

## Core Responsibilities
- Define `sound_snippet` struct for specifying audio segment boundaries (start/end offsets)
- Define `song_definition` struct to organize a complete song's metadata
- Provide flag constants for song behavior (`_song_automatically_loops`)
- Define a macro (`RANDOM_COUNT`) for representing randomized chorus repetition counts
- Declare a global `songs[]` array as the engine's song registry

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_snippet` | struct | Marks audio segment boundaries with 32-bit start/end offsets |
| `song_definition` | struct | Complete song metadata: flags, three segments (intro/chorus/trailer), chorus count, restart delay |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `songs[]` | `song_definition[]` | global | Registry of all music tracks available to the engine |

## Key Functions / Methods
None. This is a pure data-definition header.

## Control Flow Notes
This file does not participate in control flow. It serves as a data-only resource: the `songs[]` array is likely read during engine initialization to populate a music system's state machine, then queried during playback to determine segment transitions (intro ΓåÆ repeating chorus ΓåÆ trailer).

## External Dependencies
- **External symbols used**: `MACHINE_TICKS_PER_SECOND` (macro, likely from platform/timing header)
- **Type assumptions**: `int16`, `int32` (platform-dependent integer types, likely from a common header)
- **Standard**: C89/C99 struct definitions with fixed-width integer types

## Notes
- The `RANDOM_COUNT(x)` macro negates a value: used to signal that `chorus_count` should be interpreted as a random range rather than a literal count. This is a convention, not enforced by the type system.
- The sample `songs[0]` entry is placeholder data (all offsets are 0); actual song definitions would vary.
- No validation of segment offsets or counts at compile time; relies on runtime checks elsewhere.
