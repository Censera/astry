# Diagnostics

Astry defines error categories for input, options, target support, lexing, parsing, semantic analysis, and backend lowering. The current formatting is implemented in `src/utility/error.rs`; the ideal format and the rendered semantic-span format are not fully aligned yet.

## Error categories

| Error | Meaning |
|---|---|
| `Io` | Source file or another input operation failed |
| `InvalidOptions` | Compiler input or options are invalid |
| `UnsupportedTarget` | The backend does not support the requested target |
| `Lex` | Source text could not be tokenized |
| `Parse` | Tokens do not form valid syntax or structure |
| `Semantic` | A parsed construct violates language meaning or type rules |
| `Backend` | A semantic program could not be lowered to the backend |
| `StageNotImplemented` | Compilation reached a stage that is not implemented |

Lexer and parser diagnostics carry a line and column. Semantic diagnostics carry a `SourceSpan`. `with_source` attaches the source name to an error.

## Current output

The CLI prints an error to standard error and exits with status 1. A source-wrapped lexer/parser diagnostic is currently formatted along these lines:

```text
E [src/main.as][3][8] unexpected token
```

Semantic errors use a start and end span, for example:

```text
E [src/main.as][3:8-3:12] type mismatch
```

The exact message depends on the error and source. This is a description of the current implementation, not a finalized user-facing diagnostic specification.

## Useful reports

When reporting a compiler issue, include:

1. The smallest source file that reproduces it.
2. The command used.
3. The full diagnostic text.
4. The expected behavior and the observed behavior.
5. The repository branch or commit used.

For a lexer or parser issue, include the exact token or syntax that triggered the failure. For a semantic or backend issue, also include the relevant type declarations and surrounding function.

## Known gap

`doc/syntax.md` shows semantic diagnostics using `[Line][Column]`, while the current `Error` display uses a range such as `[start_line:start_column-end_line:end_column]`. `todo/utility.md` tracks this mismatch. The CLI also lacks separate exit codes for usage, source, semantic, backend, and toolchain failures.
