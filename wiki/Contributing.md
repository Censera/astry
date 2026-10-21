# Contributing

Keep changes narrow and keep the implementation, specification, tests, and documentation in agreement.

## Before changing syntax

1. Read the relevant section in [`doc/syntax.md`](../doc/syntax.md).
2. Check whether the behavior is stable or marked with an open question (`Q1` through `Q20`).
3. If the rule is unresolved, decide and document the rule before depending on it in multiple compiler stages.
4. Add or update tests for accepted syntax, rejected syntax, and source locations where relevant.
5. Update the lexer/parser/semantic TODO when the implementation status changes.

## Stage boundaries

The compiler is intended to pass structured information forward:

- The lexer owns tokenization and lexical diagnostics.
- The parser owns syntactic structure and produces the AST.
- Semantic analysis resolves names and types and validates language rules.
- The backend lowers a checked semantic program to LLVM IR.
- Output handling emits the requested artifact.

Do not compensate for a missing invariant in a later stage by reparsing source text or silently recovering from an earlier-stage failure.

## Development commands

```console
cargo build
cargo test
```

Build requirements include the LLVM/toolchain setup required by `an-inkwell`. A passing unit-test suite is not currently the same as a passing end-to-end compilation test: the compile entry point intentionally stops at the semantic stage while that stage remains unimplemented.

## Tests to add

For lexer changes, test tokens and malformed inputs. For parser changes, test both AST shape and rejection cases. For semantic changes, test name resolution, types, overload selection, visibility, and deterministic source spans. For backend changes, verify generated LLVM IR and add end-to-end tests once the frontend-to-backend path exists.

## Documentation style

- Use plain, exact language.
- Describe actual behavior, not intended behavior, as current behavior.
- Keep examples small and runnable when the toolchain supports them.
- State preconditions, failures, and unresolved behavior explicitly.
- Prefer one sentence per source line in documentation files.
- Link to the source specification instead of copying the entire grammar into several places.
- Avoid introducing architectural layers for features the language does not yet require.

## Pull requests

Keep each change focused. Explain the behavior changed, the tests run, and any known limitations. If a language rule changes, update the syntax reference and related examples in the same change.
