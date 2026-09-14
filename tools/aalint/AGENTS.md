# AGENTS.md

## Codebase Discovery

This project may use codebase-memory-mcp. Prefer graph tools for code discovery when available:

1. `search_graph` to find functions, classes, variables, and routes.
2. `trace_path` to inspect callers and callees.
3. `get_code_snippet` to read exact function or class source.
4. `query_graph` for complex graph queries.
5. `get_architecture` for high-level structure.

Use `rg` or normal file reads for strings, configs, docs, generated fixtures, or when graph results are insufficient.

## aalint Local Context

- The user runs aalint from `D:\tools\aardio\` so it can load all normal aardio libraries.
- For uncertain aardio library behavior, read `D:\tools\aardio\lib\` directly.
- After code changes, verify with `D:\tools\aardio\aalint.exe`; when publishing is needed, prefer `D:\tools\aardio\aalint.exe --ide-publish-refresh` so the user does not need to compile/copy manually.
- Current improvement order: fix correctness bugs first, then performance, then architecture.
- Keep tests focused on real aardio syntax traps and executable behavior.

## aardio Syntax Memory

- Only `false`, `null`, and `0` are falsey. Empty strings, arrays, and tables are truthy. Prefer `===` / `!==` for boolean and null checks.
- `+` is numeric addition; use `++` for string concatenation.
- A multi-return function in the last argument position expands into multiple arguments. Wrap it in parentheses to force one value.
- `try/catch` behaves like an immediately invoked anonymous function. `return`, `break`, and `continue` inside it do not exit the outer function or loop.
- Single-quoted strings process escapes. Double-quoted and backtick strings are raw/verbatim; use doubled `""` inside double-quoted strings for a literal quote.
- aardio patterns are not PCRE. `()` captures only and cannot be quantified; use `<>` for non-capturing subpatterns, remembering that it is atomic.
- `a ? b : c` does not preserve falsey `b`; use explicit `if/else` when false/null/0 are valid results.
- `?.`, JS spread, arrow functions, Lua `pairs`/`ipairs`/`pcall`, `finally`, and `object:method()` are not aardio idioms.
- `for(i, v in tab)` gives key/index then value. A single variable receives the key, not the value.
- Each thread has independent globals and imports. Import required libraries inside thread entry functions.
- `thread.invoke(fn())` passes a return value; use `thread.invoke(fn, arg1, arg2)`.
- `object.method()` passes `owner`; extracting the method or calling through `object["method"]()` can lose owner semantics.
- Arrays are 1-based. `#` is reliable for dense ordered arrays, not sparse tables or arrays containing `null`.

See `docs/aardio-syntax-traps.md` for the fuller checklist and lint-rule candidates.
