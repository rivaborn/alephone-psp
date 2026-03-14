# Source_Files/confpaths.h

## File Purpose
A compile-time configuration header that defines the installation path for package data. Used to locate shared game assets, resources, and configuration files at runtime.

## Core Responsibilities
- Define the package data directory path macro for the Aleph One game engine
- Provide a single point of configuration for asset directory location across the build

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PKGDATADIR` | String literal (macro) | Global | Absolute path to the PSP development package data directory |

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This is a compile-time configuration header with no runtime control flowΓÇöit is `#include`d by other source files to inject the data path constant during compilation.

## External Dependencies
- None. Pure preprocessor definition; no external symbols referenced.

## Notes
- The hardcoded path (`/usr/local/pspdev/psp/share/AlephOne`) suggests this targets PSP (PlayStation Portable) development.
- Likely generated or templated during the build process (e.g., via autoconf/CMake), as paths vary across installations.
- Files including this header will use `PKGDATADIR` to locate game data at runtime.
