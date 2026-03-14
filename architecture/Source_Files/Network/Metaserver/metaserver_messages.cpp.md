# Source_Files/Network/Metaserver/metaserver_messages.cpp

## File Purpose
Implements serialization/deserialization (deflation/inflation) for metaserver protocol messages in Aleph One's networking layer. Handles encoding/decoding of login credentials, room listings, player info, game descriptions, and chat messages for communication with a central metaserver. Includes platform detection, encryption negotiation, and player color management.

## Core Responsibilities
- Serialize message types to network byte streams (deflate) and deserialize from byte streams (inflate) for transmission
- Provide stream utilities for reading/writing null-terminated strings, padded data, and numeric types
- Implement 13+ message type handlers (login, room management, player lists, game listings, chat, broadcasts)
- Manage player color data with support for custom metaserver color overrides
- Parse and encode game descriptions including scenario IDs, map checksums, physics, and netscript references
- Define static lookup tables for room names, game types, and difficulty levels used in display formatting
- Handle platform detection and encryption scheme negotiation (plaintext, simple, roomserver-based)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sRoomNames | const char* array (30 elements) | file-static | Predefined names of metaserver rooms ("Crows Bridge", "Otter Ferry", etc.) |
| kRoomNameCount | const int | file-static | Size of sRoomNames array; enables bounds checking for room ID lookups |
| kServiceName | const char* | file-static | Protocol service identifier string "MARATHON" sent to metaserver |
| gameTypeString | const char* array (9 elements) | file-static | Display names for game types (EMFH, Cooperative, CTF, KOTH, etc.) |
| difficultyLevelString | const char* array (5 elements) | file-static | Display names for difficulty levels (Kindergarten through Total Carnage) |

## Key Functions / Methods

### write_player_aux_data
- Signature: `void write_player_aux_data(AOStream& out, string name, const string& team, bool away, const string& away_message)`
- Purpose: Serializes player metadata (colors, status, name, team) to network stream using metaserver-specific format and color preferences
- Inputs: output stream, player name, team name, away status, away message prefix
- Outputs/Return: void; writes 40+ bytes plus string data to stream
- Side effects: reads from global `network_preferences` and `player_preferences` to determine color source; modifies output stream position
- Calls: `get_metaserver_player_color()`, `write_padded_bytes()`, `write_string()`
- Notes: Prepends away message prefix to player name when away=true; uses custom metaserver colors if configured, otherwise falls back to player color and team color prefs; writes fixed-format color fields (3x uint16) and state flags

### LoginAndPlayerInfoMessage::reallyDeflateTo
- Signature: `void LoginAndPlayerInfoMessage::reallyDeflateTo(AOStream& thePacket) const`
- Purpose: Serializes login credentials and player profile data into metaserver login message packet
- Inputs: output stream; uses member fields m_userName, m_playerName, m_teamName
- Outputs/Return: void; encodes ~100+ bytes (platform, version, dates, user info, player aux data)
- Side effects: writes to output stream; computes player data size from member strings
- Calls: `write_padded_string()`, `write_player_aux_data()`
- Notes: Hardcodes platform detection (Windows/Mac/Other) via preprocessor; embeds __DATE__ and __TIME__; player data size calculated as 40 + strlen(playerName) + strlen(teamName); sets kResetPlayerData flag and kCRYPT_SIMPLE auth type

### operator>>(AIStream& stream, GameDescription& desc)
- Signature: `AIStream& operator>>(AIStream& stream, GameDescription& desc)`
- Purpose: Deserializes game configuration from metaserver protocol bytes into GameDescription struct; handles versioned plugin data
- Inputs: input stream; populates desc member fields (type, timeLimit, mapChecksum, difficulty, maxPlayers, etc.)
- Outputs/Return: reference to input stream for chaining; modifies desc in-place
- Side effects: reads from stream; may allocate strings via read_padded_string(); sets m_hasGameOptions, m_closed, m_running flags
- Calls: `read_padded_string()`, `read_string()`, stream ignore() for alignment/padding
- Notes: Reads 512-byte plugin data block conditionally based on pluginFlag (0x1 = scenario/build info, 0x2 = game options); if pluginFlag & 0x2, reads game options (options, cheatFlags, killLimit) and map/physics filenames; status flags decoded as bitwise operations (0x0010=closed, 0x0001=running); maxTeams set to -1 when teams disabled

### RoomListMessage::reallyInflateFrom
- Signature: `bool RoomListMessage::reallyInflateFrom(AIStream& inStream)`
- Purpose: Deserializes a list of available metaserver rooms from received message stream
- Inputs: input stream containing variable number of room entries
- Outputs/Return: bool (always true on successful read until EOF)
- Side effects: populates m_rooms vector; reads RoomDescription objects until stream exhausted
- Calls: `RoomDescription::read()` on each room; stream tellg(), maxg() for position/length checks
- Notes: Loop-based parsing using stream position tracking; no predefined room count, relies on stream bounds; each RoomDescription reads ~14 bytes (ID, player count, address/port, game count, type, padding)

## Control Flow Notes
This file is invoked asynchronously during network communication, not in a traditional game loop. When outgoing messages are prepared, the engine calls `Message::deflate()` (which invokes `reallyDeflateTo()`). When incoming messages arrive, the TCPMess framework calls `Message::inflateFrom()` (which invokes `reallyInflateFrom()`). Room lists and game lists are parsed on-demand when the player browses servers. Player color data is accessed each time a player profile is transmitted. No initialization or shutdown is required beyond including this file and linking message prototypes to the message inflater registry.

## External Dependencies
- **Message framework**: Message.h, SmallMessageHelper base class; CommunicationsChannel.h, MessageDispatcher.h for message routing
- **Stream I/O**: AStream.h (AIStream, AOStream) for byte serialization
- **Game data**: Scenario.h (Scenario::instance() for scenario ID/name/version); map.h (TICKS_PER_SECOND for time calculations); network.h (kNetworkSetupProtocolID)
- **Preferences**: preferences.h (network_preferences, player_preferences); shell.h (_get_player_color, get_player_color)
- **UI/Display**: network_dialogs.h, TextStrings.h (TS_GetCString() for localized game type strings)
- **Utilities**: boost/algorithm/string/predicate.hpp (ends_with); SDL_net.h (IPaddress, SDL_GetTicks)
- **Logging**: Logging.h (logging facilities, conditional on DISABLE_NETWORKING guard)
