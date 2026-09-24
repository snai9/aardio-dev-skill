---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'd9f8bf50-aa4f-4fe9-a7a8-be6bb27da0ee'
  PropagateID: 'd9f8bf50-aa4f-4fe9-a7a8-be6bb27da0ee'
  ReservedCode1: 'b2d319ed-9780-4850-a57d-77fc1dd2bb4d'
  ReservedCode2: 'b2d319ed-9780-4850-a57d-77fc1dd2bb4d'
---

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

- The user runs aalint from the aardio install directory (`<aardio>\`, resolve via the `AARDIO_HOME` env var or `process.aardio.getDir()`) so it can load all normal aardio libraries.
- For uncertain aardio library behavior, read `<aardio>\lib\` directly.
- After code changes, verify with `<aardio>\aalint.exe`; when publishing is needed, prefer `<aardio>\aalint.exe --ide-publish-refresh` so the user does not need to compile/copy manually.
- Current improvement order: fix correctness bugs first, then performance, then architecture.
- Keep tests focused on real aardio syntax traps and executable behavior.

## aardio Syntax Memory

- Treat the aardio compiler/runtime as the syntax authority. Prefer `loadcode()` / actual execution over recreating a parser in aalint.
- Only `false`, `null`, and `0` are falsey. Empty strings, arrays, and tables are truthy. Prefer `===` / `!==` for boolean and null checks when exact type/value matters.
- `++` is the unambiguous string-concatenation operator, but `+` can also concatenate in legal aardio expressions involving quoted string literals. Do not lint `+` as a string bug from syntax alone.
- A multi-return function in the last argument position expands into multiple arguments. Wrap it in parentheses to force one value.
- `try/catch` behaves like an immediately invoked anonymous function. `return` only returns from that try/catch scope; static detection of the exact scope is heuristic.
- Single-quoted strings process escapes. Double-quoted and backtick strings are raw/verbatim; all three forms may span source lines with different newline semantics.
- `//` and variable-star `/* ... */` forms can also participate in aardio raw-text/string syntax depending on context. For lint masking, both comment/raw-text forms are non-code.
- Template syntax is enabled when the first non-whitespace source token is `<?` or `?>`; outside `<? ... ?>` is raw template text. `<?xml` is not an aardio code-open marker.
- aardio patterns are not PCRE. `string.find/match/replace` use aardio pattern semantics unless a literal API such as `string.indexOf` is intended.
- `a ? b : c` does not preserve falsey `b`; however official code intentionally uses this behavior, so it should not be a default lint warning.
- Do not infer JavaScript optional chaining solely from the byte sequence `?.`; official aardio source contains legal operator combinations such as `#?..`. Likewise `:` is a valid aardio operator, so `object:method()`-looking text is not safe to lint by substring alone.
- Generic `for ... in ...` consumes iterator return values. A single loop variable receives the iterator's first return value; it is a key only for iterators whose first return is a key (such as direct table iteration).
- Each thread has independent globals/imports. Import required libraries inside thread entry functions when needed.
- `thread.invoke(fn())` passes the return value of `fn`; use `thread.invoke(fn, arg1, arg2)` when the function itself is intended.
- `ioFileObject.write()` returns the file object on success, so chaining `.write(...).close()` is legal.
- In an unambiguous equality expression aardio may accept single `=` as `==`; treat it as style/experimental, not a generic assignment-in-condition error.
- `table.isArray()` is a valid exact pure-array test. Do not automatically rewrite it to `table.isArrayLike()`.
- `object.method()` passes `owner`; extracting the method or calling through `object["method"]()` can lose owner semantics.
- Arrays are 1-based. `#` is reliable for dense ordered arrays, not sparse tables or arrays containing `null`.

See `docs/aardio-syntax-traps.md` for the fuller checklist and lint-rule candidates.

> AI生成