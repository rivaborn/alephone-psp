# Source_Files/Misc/Logging.h

## File Purpose
Core logging infrastructure for the Aleph One game engine, providing a flexible multi-level logging system with domain-based filtering, context stacks, and thread-safe variants. Supports 8 severity levels from Fatal to Dump, with customizable output behavior per domain.

## Core Responsibilities
- Define logging severity levels (Fatal, Error, Warning, Anomaly, Note, Summary, Trace, Dump)
- Provide abstract Logger base class for pluggable logging implementations
- Supply convenience macros for main-thread and non-main-thread logging (logError, logWarning, etc.)
- Implement stack-based logging context for hierarchical log grouping
- Enable per-domain configuration (threshold, file/line display, output flushing)
- Generate variadic macro variants via Logging_gruntwork.h for flexible argument counts

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Logger | class (abstract) | Pluggable logging backend; subclasses implement actual log output |
| LogContext | class | RAII wrapper for stack-based log context entry/exit |
| LogAction | struct (nested in commented SubLogger) | Deferred log event record (message, context push/pop); not yet integrated |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| logDomain | const char* extern | global | Default domain for logging when caller doesn't specify |

## Key Functions / Methods

### Logger::pushLogContext
- **Signature:** `virtual void pushLogContext(const char* inFile, int inLine, const char* inContext, ...)`
- **Purpose:** Begin a named logging context with optional printf-style formatting
- **Inputs:** File/line location, context description string, variadic args
- **Outputs/Return:** None
- **Side effects:** Pushes context onto internal stack; may be thread-aware
- **Calls:** (delegated to pushLogContextV by wrapper)
- **Notes:** Main-thread only unless wrapper forwards to logMessageNMT instead

### Logger::logMessage
- **Signature:** `virtual void logMessage(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- **Purpose:** Log a message at specified severity level to named domain
- **Inputs:** Domain name, severity level constant, caller location, message + variadic args
- **Outputs/Return:** None
- **Side effects:** Routes to logMessageV; output buffered/flushed per domain config
- **Calls:** (delegated to logMessageV by wrapper)
- **Notes:** Main-thread only; macros (logError, logWarning, etc.) wrap this

### Logger::logMessageNMT
- **Signature:** `virtual void logMessageNMT(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- **Purpose:** Non-main-thread variant of logMessage; may have simplified/safer implementation
- **Inputs:** Same as logMessage
- **Outputs/Return:** None
- **Side effects:** Thread-safe logging; implementation deferred to subclass
- **Notes:** Used by logErrorNMT, logWarningNMT, etc. macros

### Logger::pushLogContextV / popLogContext / logMessageV
- **Purpose:** va_list variants; pure virtual, implemented by subclasses
- **Notes:** Subclasses override these to handle variadic unpacking and actual output

### GetCurrentLogger
- **Signature:** `Logger* GetCurrentLogger()`
- **Purpose:** Retrieve the active Logger instance (singleton-like accessor)
- **Outputs/Return:** Pointer to current Logger
- **Notes:** Implementation not in this file; likely set globally at startup

### LogContext::enterContextV / leaveContext
- **Purpose:** Push/pop logging context; platformaware (early Mac Classic vs. Mach)
- **Notes:** Conditional compilation skips context push/pop on non-Mach Mac for non-main threads; otherwise always active

### Logging_GetParser / setLoggingThreshhold / setShowLoggingLocations / setFlushLoggingOutput
- **Purpose:** Configuration accessors for XML parser, per-domain filtering, output control
- **Notes:** Implementation deferred to .cpp; enable runtime tweaking without Logger subclass

## Control Flow Notes
**Logging Context Stack:** LogContext (RAII) calls `GetCurrentLogger()->pushLogContextV()` on construction and `popLogContext()` on destruction, enabling indented/nested log messages. The stack is domain-specific and supports variadic context descriptions.

**Macro Delegation:** Convenience macros (logError, logWarning, logContext, etc.) in Logging_gruntwork.h forward to Logger methods with `__FILE__`, `__LINE__`, and domain prepopulated. Macros exist in two variants:
- **Main-thread** (logError, logContext, etc.): call logMessage / pushLogContext
- **Non-main-thread** (logErrorNMT, logContextNMT, etc.): call logMessageNMT; context behavior platform-gated

**Severity Levels:** 0 (Fatal, hard stop) ΓåÆ 60 (Dump, verbose data); logger filters at message level < threshold per domain.

## External Dependencies
- `<stdarg.h>` ΓÇö variadic argument handling (va_list, va_start, va_end)
- `Logging_gruntwork.h` ΓÇö auto-generated convenience macro definitions (logError, logWarning1, logContext3, etc. for 0ΓÇô5 args)
- `XML_ElementParser` ΓÇö forward-declared; used by Logging_GetParser for config parsing (defined elsewhere)
- `logDomain` extern ΓÇö catch-all domain string (defined elsewhere)
