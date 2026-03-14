# Source_Files/Misc/Logging.cpp

## File Purpose
Implements a hierarchical logging system for Aleph One with context stacks, configurable output filtering, and XML-based configuration. Logs are written to platform-specific locations (macOS Library/Logs or Unix ~/.alephone/) with optional file locations and severity levels.

## Core Responsibilities
- **Logger initialization**: Lazy-initialize a singleton TopLevelLogger on first use; create log file in platform-appropriate location
- **Context stack management**: Push/pop logging contexts to build hierarchical indentation in output
- **Message filtering**: Filter log messages by severity level (threshold); only emit below threshold
- **Output control**: Optionally show source file/line numbers; optionally flush after every write
- **XML configuration**: Parse `<logging_domain>` elements from config files to configure thresholds, locations, and flushing per domain
- **Thread safety**: Provide non-main-thread (NMT) variants that safely disable logging on Mac OS 9

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TopLevelLogger` | class | Concrete Logger implementation; maintains context stack and outputs formatted messages to file |
| `XML_LoggingConfigurationParser` | class | Extends XML_ElementParser; parses `<logging_domain>` attributes (threshhold, show_locations, flush) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sCurrentLogger` | `Logger*` | static | Singleton pointer to active logger; lazily initialized |
| `sOutputFile` | `FILE*` | static | Open file handle for log output; opened in InitializeLogging() |
| `sLoggingThreshhold` | `int` | static | Messages with level < threshold are logged; default logNoteLevel |
| `sShowLocations` | `bool` | static | If true, append source file and line number to log messages |
| `sFlushOutput` | `bool` | static | If true, fflush() after each message (safety for expected crashes) |
| `logDomain` | `const char*` | extern | Global logging domain identifier ("global") |
| `LoggingConfigurationParser` | `XML_LoggingConfigurationParser` | static | Reusable XML parser for logging config elements |
| `LoggingParser` | `XML_ElementParser` | static | Root XML parser; children include LoggingConfigurationParser |

## Key Functions / Methods

### GetCurrentLogger()
- **Signature:** `Logger* GetCurrentLogger()`
- **Purpose:** Obtain the active Logger instance; triggers one-time initialization on first call
- **Inputs:** None
- **Outputs/Return:** Pointer to singleton TopLevelLogger
- **Side effects:** Calls InitializeLogging() if `sCurrentLogger == NULL`; creates output file and logger instance
- **Calls:** `InitializeLogging()`

### TopLevelLogger::pushLogContextV()
- **Signature:** `void pushLogContextV(const char* inFile, int inLine, const char* inContext, va_list inArgs)`
- **Purpose:** Format and push a context string onto the context stack
- **Inputs:** source file, line number, format string, variadic args
- **Outputs/Return:** None (modifies `mContextStack`)
- **Side effects:** Appends formatted context (with optional location) to mContextStack
- **Calls:** `vsnprintf()`, `snprintf()`
- **Notes:** If sShowLocations is true, appends " (file:line)" to context string

### TopLevelLogger::popLogContext()
- **Signature:** `void popLogContext()`
- **Purpose:** Remove the most recent context from the stack; track common depth
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `mContextStack.pop_back()`; updates `mMostRecentCommonStackDepth` if stack shrinks
- **Calls:** None (direct vector operations)

### TopLevelLogger::logMessageV()
- **Signature:** `void logMessageV(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, va_list inArgs)`
- **Purpose:** Core logging function; format message and emit to file with context indentation
- **Inputs:** domain, severity level, source file, line number, format string, variadic args
- **Outputs/Return:** None
- **Side effects:** Writes to `sOutputFile` (fprintf); updates `mMostRecentCommonStackDepth` and `mMostRecentlyPrintedStackDepth`
- **Calls:** `vsnprintf()`, `snprintf()`, `fprintf()`, `fflush()`
- **Notes:** Only logs if `sOutputFile != NULL && inLevel < sLoggingThreshhold`; outputs context stack with indent (2 spaces per depth), then message, then optional location

### InitializeLogging()
- **Signature:** `static void InitializeLogging()`
- **Purpose:** One-time setup of logging file and TopLevelLogger singleton
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Opens log file (macOS: `~/Library/Logs/Aleph One Log.txt`, Unix: `~/.alephone/Aleph One Log.txt`, fallback: `./Aleph One Log.txt`); allocates and assigns `sCurrentLogger`; writes timestamp separator
- **Calls:** `getenv()`, `getpwuid()`, `fopen()`, `time()`, `ctime()`, `fprintf()`
- **Notes:** Asserts `sOutputFile == NULL` on entry; uses POSIX pwd.h on Unix/macOS; appends to existing file

### setLoggingThreshhold(), setShowLoggingLocations(), setFlushLoggingOutput()
- **Signature:** `void setLoggingThreshhold(const char* inDomain, int16 inThreshhold)` (and analogues)
- **Purpose:** Modify global logging configuration (currently domain parameter unused)
- **Inputs:** domain (ignored), configuration value
- **Outputs/Return:** None
- **Side effects:** Modifies `sLoggingThreshhold`, `sShowLocations`, or `sFlushOutput`; setFlushLoggingOutput also calls `fflush()` immediately if enabled
- **Calls:** `fflush()`

### XML_LoggingConfigurationParser::HandleAttribute()
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse XML attributes (domain, threshhold, show_locations, flush)
- **Inputs:** attribute tag name, attribute value string
- **Outputs/Return:** `true` on success, `false` on error (sets `ErrorString`)
- **Side effects:** Populates member fields; validates no attribute is specified twice
- **Calls:** `StringsEqual()`, `ReadInt16Value()`, `ReadBooleanValueAsBool()`, `UnrecognizedTag()`
- **Notes:** Case-insensitive tag matching; rejects multiply-specified attributes

### XML_LoggingConfigurationParser::AttributesDone()
- **Signature:** `bool AttributesDone()`
- **Purpose:** Finalize parsing and apply configuration if any attributes were present
- **Inputs:** None (uses member `mAttributePresent[]` and values)
- **Outputs/Return:** `true` on success, `false` if domain missing when attributes present
- **Side effects:** Calls `setLoggingThreshhold()`, `setShowLoggingLocations()`, and/or `setFlushLoggingOutput()` based on which attributes were parsed
- **Calls:** Configuration setter functions

### Logging_GetParser()
- **Signature:** `XML_ElementParser* Logging_GetParser()`
- **Purpose:** Provide the root XML parser for logging configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to static `LoggingParser` (with LoggingConfigurationParser attached as child)
- **Side effects:** Attaches LoggingConfigurationParser as a child on first call (not idempotent)
- **Calls:** `AddChild()`

## Control Flow Notes
- **Initialization**: Lazy on first `GetCurrentLogger()` call ΓåÆ `InitializeLogging()` creates file and singleton
- **Logging flow**: App calls `logMessage()` macro ΓåÆ `Logger::logMessage()` ΓåÆ `logMessageV()` ΓåÆ `TopLevelLogger::logMessageV()` ΓåÆ writes to file if threshold met
- **Context flow**: App pushes context (via `logContext()` macro or `LogContext` RAII class) ΓåÆ stack grows ΓåÆ subsequent logs indent by 2├ù stack depth
- **Configuration**: XML parser reads `<logging>` elements, extracts `<logging_domain>` children, applies settings via setter functions
- **Shutdown**: No explicit cleanup; file remains open; relies on process exit

## External Dependencies
- **Standard C/C++**: `<stdarg.h>`, `<fstream>`, `<string>`, `<vector>`, `<ctime>`, `<cstdio>`
- **Platform-specific**: `<unistd.h>`, `<sys/types.h>`, `<pwd.h>` (Unix/macOS only)
- **Custom**: `cseries.h` (utility macros), `XML_ElementParser.h` (base parser class), `snprintf.h` (platform compatibility)
- **Conditional**: `snprintf.h` only if `!HAVE_SNPRINTF`
