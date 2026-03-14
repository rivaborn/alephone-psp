# Source_Files/Network/network_distribution_types.h

## File Purpose
Defines enumeration constants for network audio distribution type IDs in the Aleph One game engine. Centralizes distribution type identifiers to avoid conflicts across the codebase and maintains backward compatibility with older versions.

## Core Responsibilities
- Define network audio distribution type identifiers for use in `NetDistributeInformation` and `NetAddDistributionFunction`
- Maintain a registry for legacy vs. modern audio streaming protocols
- Provide a centralized, conflict-free location for distribution type constants

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kOriginalNetworkAudioDistributionTypeID` | `int` (enum value 0) | Global | Legacy network audio distribution identifier for backward compatibility |
| `kNewNetworkAudioDistributionTypeID` | `int` (enum value 1) | Global | Modern real-time network audio distribution identifier |

## Key Functions / Methods
None.

## Control Flow Notes
This is a purely declarative header file with no control flow. It defines constants used by network distribution code elsewhere in the engine.

## External Dependencies
- Standard C header guards (`#ifndef`, `#define`, `#endif`)
- No external includes or symbol dependencies
