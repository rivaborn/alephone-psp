# Source_Files/Lua/llex.h

## File Purpose

Lexical analyzer header for Lua. Defines token types, the LexState structure that maintains tokenization state, and the public API for lexing source code into a stream of tokens consumed by the parser.

## Core Responsibilities

- Define all token types (reserved keywords, operators, literals, special tokens)
- Provide the LexState structure to track position, current/lookahead tokens, and input streams
- Define SemInfo union to store semantic data for tokens (numeric constants or interned strings)
- Declare the lexer initialization and tokenization API (luaX_next, lookahead, error reporting)
- Manage string interning via luaX_newstring to deduplicate string values

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| RESERVED | enum | All token types: reserved words (TK_AND, TK_FUNCTION, etc.), operators (TK_CONCAT, TK_EQ), literals (TK_NUMBER, TK_STRING, TK_NAME), and special (TK_EOS) |
| SemInfo | union | Semantic value for a tokenΓÇöeither a lua_Number or a TString* pointer |
| Token | struct | Single token: type (int token) and semantic info (SemInfo seminfo) |
| LexState | struct | Complete lexer state: current character, line numbers, current/lookahead tokens, input stream (ZIO), memory buffer, source name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| luaX_tokens | const char * const [] | global | Array of token names indexed by token type; defined elsewhere |

## Key Functions / Methods

### luaX_init
- Signature: `void luaX_init(lua_State *L)`
- Purpose: Initialize the lexer module
- Inputs: Lua state
- Outputs/Return: void
- Side effects: Sets up internal tables (likely token name mappings)
- Calls: Not inferable from this file
- Notes: Called once at startup

### luaX_setinput
- Signature: `void luaX_setinput(lua_State *L, LexState *ls, ZIO *z, TString *source)`
- Purpose: Initialize a LexState for a new input stream; resets all tokenization state
- Inputs: Lua state, LexState pointer (output), input stream ZIO, source file name
- Outputs/Return: void (modifies ls in place)
- Side effects: Initializes LexState fields (current char, line number, buffers)
- Calls: Not inferable from this file
- Notes: Must be called before luaX_next on a fresh LexState

### luaX_newstring
- Signature: `TString *luaX_newstring(LexState *ls, const char *str, size_t l)`
- Purpose: Intern a string; returns existing TString if already interned, otherwise allocates new one
- Inputs: LexState (for memory context), string buffer, length
- Outputs/Return: Interned TString pointer
- Side effects: May allocate memory via Lua's allocator
- Calls: Not inferable from this file
- Notes: Enables pointer equality for identical strings

### luaX_next
- Signature: `void luaX_next(LexState *ls)`
- Purpose: Advance to the next token, updating LexState.t with the new token
- Inputs: LexState
- Outputs/Return: void (result in ls->t)
- Side effects: Advances input stream, updates current character, line number, token buffer
- Calls: Not inferable from this file
- Notes: Core tokenization function; called repeatedly by parser

### luaX_lookahead
- Signature: `void luaX_lookahead(LexState *ls)`
- Purpose: Peek at the next token without consuming the current one
- Inputs: LexState
- Outputs/Return: void (result in ls->lookahead)
- Side effects: May advance internal buffering to read ahead
- Calls: Not inferable from this file
- Notes: Used for predictive parsing; allows parser to make decisions based on next token

### luaX_lexerror
- Signature: `void luaX_lexerror(LexState *ls, const char *msg, int token)`
- Purpose: Report a lexical (tokenization) error with source location
- Inputs: LexState (for line info), error message, optional token involved
- Outputs/Return: void (terminates execution or throws)
- Side effects: Error reporting/handling
- Calls: Not inferable from this file
- Notes: Called when an invalid token or character is encountered

### luaX_syntaxerror
- Signature: `void luaX_syntaxerror(LexState *ls, const char *s)`
- Purpose: Report a syntax error (parser-level error) with source location
- Inputs: LexState (for line info), error description
- Outputs/Return: void (terminates execution or throws)
- Side effects: Error reporting/handling
- Calls: Not inferable from this file
- Notes: Called by parser when token sequence is invalid

### luaX_token2str
- Signature: `const char *luaX_token2str(LexState *ls, int token)`
- Purpose: Convert a token type constant to its human-readable name
- Inputs: LexState, token type code
- Outputs/Return: Pointer to token name string
- Calls: Not inferable from this file
- Notes: Used in error messages to display token names to users

## Control Flow Notes

**Initialization**: Parser calls `luaX_setinput()` to attach a LexState to an input stream.

**Main loop**: Parser repeatedly calls `luaX_next()` to consume tokens. Between calls, `ls->t` holds the current token, and `luaX_lookahead()` populates `ls->lookahead` for one-token-ahead decisions.

**Line tracking**: LexState maintains `linenumber` (current line) and `lastline` (line of last token consumed), used for error reporting.

**Buffering**: The `buff` (Mbuffer) accumulates partial tokens (e.g., long string literals or multiline comments) before finalizing them.

## External Dependencies

- **lobject.h**: TString (interned strings), lua_Number, basic Lua value types
- **lzio.h**: ZIO (buffered input stream), Mbuffer (dynamic buffer)
- **lua.h**: lua_State (main Lua context, included indirectly)
- **luaX_tokens[]**: Token name table, defined in llex.c (implementation file)

## Notes

- Token codes begin at `FIRST_RESERVED` (257) to avoid conflicts with ASCII character codes used as single-char tokens.
- The `RESERVED` enum order is critical; grep "ORDER RESERVED" hints that the implementation relies on a specific sequence for reserved word lookup.
- `TOKEN_LEN` macro captures the longest reserved word ("function") for buffer sizing.
- `NUM_RESERVED` macro counts reserved keywords (distance from TK_WHILE to FIRST_RESERVED).
