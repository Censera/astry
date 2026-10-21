# Status and roadmap

Astry is under active development. Read the syntax document as the language design, not as a feature-completion matrix.

## Current state observed in the repository

| Area | State |
|---|---|
| Rust crate and command-line entry point | Present; CLI accepts one source-file argument |
| Lexer | Implemented with tokenization tests; lexer TODO still lists token conformance and operator cleanup |
| Parser | Program/declaration parsing and some expression parsing exist; implementation and tests do not establish full v26 conformance |
| Macro expansion and function-return normalization | Implemented as token-stage helpers used before program parsing |
| Semantic analysis | Data structures and type-syntax groundwork exist; the complete validation and resolution stage is unfinished |
| LLVM wrappers | Basic context/module/function/handler wrappers exist; unit tests cover small IR operations |
| Astry-to-LLVM lowering | Not complete end to end |
| Object/executable output | Not available through the current compile path |
| CLI options and polished diagnostics | Incomplete |
| README and user documentation | Now being established by these wiki pages; source specification remains in `doc/syntax.md` |

The strongest current limitation is explicit in `Compiler::compile_with_options`: after preparing the semantic program for each source, it returns `StageNotImplemented(Stage::Semantic)`. A valid program therefore cannot currently pass through the CLI to produce native output.

## Implementation order

The plan in `doc/plan.md` organizes the work roughly as follows:

1. Establish compiler input ownership, error handling, and a small public API.
2. Finish lexer conformance to the v26 token rules.
3. Complete the parser and AST for the stable language surface.
4. Implement semantic analysis: name resolution, scopes, visibility, types, overloads, casts, pointers, and control-flow rules.
5. Finalize the `an-inkwell` boundary and backend representations.
6. Lower checked programs into LLVM IR.
7. Implement runtime and standard-library operations.
8. Emit object files and executables for supported native targets.
9. Revisit experimental features after the core language works.

## Unresolved specification questions

`doc/syntax.md` marks unresolved behavior from `Q1` through `Q20`. The major groups are:

- **Declarations and modules:** default visibility, nested import binding, multiple-name binding semantics, and uninitialized storage.
- **Blocks and control flow:** final-expression return behavior, conditionals without `else`, match wildcard/exhaustiveness, and enum patterns.
- **Calls and methods:** variadic argument typing/boundaries and how `into` methods receive their receiver.
- **Types and casts:** equivalence of struct declaration forms, valid cast pairs, operator precedence/associativity, and numeric-literal default types.
- **Macros:** whether substituted arguments are parenthesized and how body braces participate in expansion.
- **Pointers and memory:** optional-pointer narrowing, lifetimes of addresses to locals, who frees vector and string storage, and nested union behavior.
- **Collections:** vector length versus capacity for uninitialized forms, vector method return types, and range slicing results.

Do not silently invent answers to these questions in code or documentation. Record a decision in `doc/syntax.md` when the language design resolves one.

## Cross-document inconsistencies

The repository contains a few design/status mismatches that should be resolved as implementation proceeds:

- `doc/design.md` and `doc/plan.md` put modules and macros among experimental features, while `doc/syntax.md` describes module/import handling and macro expansion in its main pipeline.
- `doc/goal.md` links to `syntaxdesign-26.8.md`, but the tracked syntax file is `doc/syntax.md`.
- The syntax document's example diagnostic format differs from the current `Error` display for semantic spans.
- The lexer and semantic TODOs identify operator/type-token mismatches that need conformance checks.
- `src/semantic/semantic.rs` currently spells the internal flag variants `Striped` and `Lossely`, while the syntax uses `@Stripped` and `@Loosely`; reconcile these names when flag handling is implemented.

The syntax document should remain authoritative for spelling and grammar. Other documents should be updated when a decision changes.
