# Source_Files/CSeries/csstrings.cpp

## File Purpose
Provides string manipulation utilities for the Aleph One game engine, supporting legacy Pascal strings (length-prefixed), C strings (null-terminated), and character encoding conversions between Mac Roman and Unicode/UTF-8. Bridges modern C++ string APIs with historical macOS resource-based string systems.

## Core Responsibilities
- Retrieve and manage Pascal/C strings from a resource system (TextStrings)
- Convert between Pascal strings, C strings, and modern `std::string`
- Format strings using printf-style variadic functions
- Perform Mac Roman Γåö Unicode character encoding conversions
- Support UTF-8 encoding/decoding for international text
- Provide safe string operations (bounds-checked copying, in-place conversion)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| MacRomanUnicodeConverter | class | Bidirectional conversion between Mac Roman (8-bit) and Unicode (16-bit); maintains lookup tables and a reverse mapping for non-ASCII characters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| temporary | char[256] | global | General-purpose temporary string buffer |
| mac_roman_to_unicode_table | uint16[256] | static (class member) | Lookup table: Mac Roman byte ΓåÆ Unicode code point (from Unicode.org APPLE/ROMAN mapping) |
| unicode_to_mac_roman | std::map<uint16, char> | static (class member) | Reverse lookup: Unicode code point ΓåÆ Mac Roman byte (built at runtime for non-ASCII subset) |
| initialized | bool | static (class member) | One-time initialization flag for MacRomanUnicodeConverter |
| macRomanUnicodeConverter | MacRomanUnicodeConverter | static | Singleton instance; lazy-initialized on first use |

## Key Functions / Methods

### countstr
- **Signature:** `size_t countstr(short resid)`
- **Purpose:** Count contiguous strings in a named resource set.
- **Inputs:** Resource ID
- **Outputs/Return:** Count of strings (0 if set not found)
- **Side effects:** None
- **Calls:** `TS_CountStrings`
- **Notes:** Wrapper around TextStrings API; index is 0-based.

### getpstr / getcstr
- **Signature:** `unsigned char *getpstr(unsigned char *string, short resid, size_t item)` / `char *getcstr(char *string, short resid, size_t item)`
- **Purpose:** Retrieve a string from a resource set into a caller-provided buffer.
- **Inputs:** Destination buffer, resource ID, index within set
- **Outputs/Return:** Pointer to destination buffer; filled with resource string or zero-length string if not found
- **Side effects:** Writes to destination buffer (caller responsible for size)
- **Calls:** `TS_GetString`
- **Notes:** getpstr copies Pascal format (length byte + data); getcstr offsets by 1 byte to extract C string. No bounds checking on destination.

### build_stringvector_from_stringset
- **Signature:** `const vector<string> build_stringvector_from_stringset(int resid)`
- **Purpose:** Load all strings from a resource set into a std::vector.
- **Inputs:** Resource ID
- **Outputs/Return:** Vector of strings; empty if set not found
- **Side effects:** Allocation
- **Calls:** `TS_GetCString` (iterated)
- **Notes:** Iterates from index 0 until `TS_GetCString` returns NULL.

### csprintf / psprintf
- **Signature:** `char *csprintf(char *buffer, const char *format, ...)` / `unsigned char *psprintf(unsigned char *buffer, const char *format, ...)`
- **Purpose:** Format a string into a caller-provided buffer (printf-style).
- **Inputs:** Destination buffer, format string, variadic arguments
- **Outputs/Return:** Pointer to destination buffer
- **Side effects:** Writes to buffer; psprintf also sets length byte
- **Calls:** `vsprintf`
- **Notes:** psprintf writes length byte at offset 0, C-string data at offset 1. No bounds checking.

### copy_string_to_pstring / copy_string_to_cstring
- **Signature:** `void copy_string_to_pstring(const std::string &s, unsigned char* dst, int maxlen)` / `void copy_string_to_cstring(const std::string &s, char* dst, int maxlen)`
- **Purpose:** Convert modern `std::string` to legacy string formats.
- **Inputs:** Source std::string, destination buffer, max bytes
- **Outputs/Return:** None (writes to destination)
- **Side effects:** Destination buffer modified; pstring truncated to maxlen chars
- **Calls:** `string::copy`
- **Notes:** pstring version sets length byte; cstring version null-terminates.

### pstring_to_string
- **Signature:** `const std::string pstring_to_string(const unsigned char* ps)`
- **Purpose:** Convert Pascal string to std::string.
- **Inputs:** Pascal string (length byte + data)
- **Outputs/Return:** Newly constructed std::string
- **Side effects:** Allocation
- **Calls:** None
- **Notes:** Uses reinterpret_cast; relies on length byte at offset 0.

### a1_p2cstr / a1_c2pstr
- **Signature:** `char *a1_p2cstr(unsigned char* inoutStringBuffer)` / `unsigned char *a1_c2pstr(char *inoutStringBuffer)`
- **Purpose:** In-place conversion between Pascal and C string formats.
- **Inputs:** Buffer containing source format (modified in-place)
- **Outputs/Return:** Pointer to same buffer, now in target format
- **Side effects:** Buffer reordered using `memmove`; data loss if length > 255 (c2pstr truncates)
- **Calls:** `memmove`, `strlen`
- **Notes:** a1_c2pstr truncates C strings > 255 chars. Caller must ensure buffer is large enough.

### pstrncpy
- **Signature:** `unsigned char *pstrncpy(unsigned char *dest, const unsigned char *source, size_t total_byte_count)`
- **Purpose:** Safe Pascal string copy with truncation and padding.
- **Inputs:** Destination buffer, source Pascal string, total byte budget (including length byte)
- **Outputs/Return:** Pointer to destination
- **Side effects:** Destination filled; null-padded if source is short
- **Calls:** `memcpy`, `memset`
- **Notes:** Protects against buffer overflow; destination length byte set to account for budget.

### pstrdup
- **Signature:** `unsigned char *pstrdup(const unsigned char *inString)`
- **Purpose:** Allocate and duplicate a Pascal string.
- **Inputs:** Source Pascal string
- **Outputs/Return:** Newly malloc'd buffer; must be freed by caller
- **Side effects:** Heap allocation
- **Calls:** `malloc`, `pstrcpy`

### mac_roman_to_utf8 / utf8_to_mac_roman
- **Signature:** `std::string mac_roman_to_utf8(const std::string& input)` / `std::string utf8_to_mac_roman(const std::string& input)`
- **Purpose:** Convert between Mac Roman and UTF-8 string encodings.
- **Inputs:** Source string in one encoding
- **Outputs/Return:** New string in target encoding
- **Side effects:** Allocation
- **Calls:** `mac_roman_to_unicode`, `unicode_to_mac_roman`, helper functions
- **Notes:** Handles multi-byte UTF-8 sequences; unknown characters mapped to '?'.

## Notes
- **MacRomanUnicodeConverter class:** Lazy-initializes the bidirectional lookup on first instantiation. Static lookup table covers 0x00ΓÇô0xFF; reverse map built only for bytes 0x80ΓÇô0xFF (ASCII is identity). Methods `ToUnicode()` and `ToMacRoman()` are inline.
- **Encoding helpers (unicode_to_utf8, utf8_to_unicode):** Internal static functions handling 1ΓÇô3 byte UTF-8 sequences.
- **Overloads:** `mac_roman_to_unicode` has three overloads (single char, unbounded string, bounded string); all delegate to the MacRomanUnicodeConverter singleton.
- **Deprecated functions:** `dprintf` and `fdprintf` now delegate to the Logging framework (`GetCurrentLogger()->logMessageV`); comments encourage migration to `logError`, `logWarning`, etc.

## Control Flow Notes
This is a utility library with no frame/update/render pattern. MacRomanUnicodeConverter initializes once (lazily, on first call) via the static singleton. Resource string functions delegate to the TextStrings API. Modern std::string integration functions are simple wrappers around memcpy/string member functions.

## External Dependencies
- **TextStrings.h:** `TS_GetString`, `TS_GetCString`, `TS_CountStrings` ΓÇö resource string lookup
- **Logging.h:** `GetCurrentLogger`, `logDomain`, `logAnomalyLevel` ΓÇö logging framework (dprintf/fdprintf now use this)
- **Standard library:** `<stdio.h>`, `<stdarg.h>`, `<string.h>`, `<map>`, `<string>`, `<vector>`
- **Conditional:** `<Carbon/Carbon.h>` when `EXPLICIT_CARBON_HEADER` is defined (macOS legacy support)
