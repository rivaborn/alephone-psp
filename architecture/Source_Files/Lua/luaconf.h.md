# Source_Files/Lua/luaconf.h

## File Purpose
Configuration header for the Lua interpreter that defines compile-time settings, platform-specific features, and behavioral parameters. Allows customizing Lua without modifying core source files via preprocessor macros marked with `@@` comments.

## Core Responsibilities
- Platform detection (Windows, Linux, macOS, POSIX compatibility)
- Define library search paths and module loading behavior
- Configure API visibility, linkage, and DLL export decorators
- Set memory management and garbage collection parameters
- Define numeric type representation and arithmetic operations
- Configure exception handling and error longjmp behavior
- Manage backward compatibility modes for older Lua versions
- Establish recursion depth, stack size, and variable limits
- Configure standard input handling and REPL prompts for standalone interpreter

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `luai_Cast` | union | Fast double-to-int conversion on Pentium: `double l_d; long l_l` |

## Global / File-Static State
None.

## Key Functions / Methods
None. File contains preprocessor macro definitions only.

## Control Flow Notes
This is a compile-time configuration header included early in Lua's build chain. The `@@` markers indicate user-customizable definitions. Conditional compilation (`#if`/`#ifdef`) selects platform-specific implementations:
- **Windows** (`_WIN32`): DLL export decorators, `\` path separators, `_isatty()` stdin detection
- **POSIX** (`LUA_USE_POSIX`): `mkstemp`, `isatty`, `popen`, `_longjmp`/_setjmp` 
- **Linux** (`LUA_USE_LINUX`): dlopen dynamic loading, readline support
- **macOS** (`LUA_USE_MACOSX`): dyld dynamic loading

## External Dependencies
**Conditional includes** (based on platform/build flags):
- `<limits.h>`, `<stddef.h>` ΓÇö always included
- `<unistd.h>` ΓÇö POSIX systems (isatty, mkstemp)
- `<io.h>`, `<stdio.h>` ΓÇö Windows
- `<assert.h>` ΓÇö when `LUA_USE_APICHECK` defined
- `<math.h>` ΓÇö when `LUA_CORE` defined
- `<readline/readline.h>`, `<readline/history.h>` ΓÇö when `LUA_USE_READLINE` defined

**Compiler/platform symbols used**:
- `__STRICT_ANSI__`, `__GNUC__`, `__ELF__`, `__cplusplus`, `_MSC_VER` ΓÇö compiler detection
- `__declspec(dllexport/dllimport)` ΓÇö MSVC DLL export
- `__attribute__((visibility("hidden")))` ΓÇö GCC/ELF visibility
- `INT_MAX`, `LONG_MAX`, `BUFSIZ` ΓÇö from standard headers
