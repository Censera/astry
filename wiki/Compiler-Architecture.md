# Compiler architecture

Astry's intended compiler is a direct pipeline. Each stage should own one responsibility, expose the information required by the next stage, and avoid making later stages recover information that an earlier stage already had.

## Intended pipeline

```text
source units
  -> lexer
  -> return-type normalization
  -> macro expansion
  -> declaration index
  -> parser / AST
  -> semantic analysis
  -> backend lowering
  -> LLVM IR
  -> object or executable output
```

The language reference describes the token-based stages before parsing so macro-produced declarations can be indexed before the parser needs to interpret name kinds. The broader design document gives the ownership boundary; the syntax document has the more detailed pipeline.

## Source ownership

`Source` owns a source unit's name and text. `Sources` owns the ordered set of source units and rejects duplicate names. Source order is supplied by the caller; compilation must not depend on filesystem enumeration order.

The intended ownership flow is:

```text
Source / Sources
  -> tokens
  -> AST / Program
  -> semantic program and resolved symbols/types
  -> backend objects and LLVM context
  -> emitted output
```

Later stages should consume structured data from earlier stages rather than rereading and reparsing source text. Source text remains available for diagnostics.

## Rust modules

| Path | Responsibility |
|---|---|
| `src/entry/main.rs` | Reads one source file, calls the compiler, prints errors, sets a non-zero exit status on failure |
| `src/lexer/lexer.rs` | Token types and source tokenization |
| `src/parser/parser.rs` | Parser implementation and AST-related syntax handling |
| `src/parser/program.rs` | Program-level parsing and declarations |
| `src/parser/parser_contract.rs` | Normalizes the function return-type spelling before parsing |
| `src/parser/macro_expand.rs` | Token-based macro expansion |
| `src/semantic/` | Semantic-program representation, type syntax, user types, source spans; full semantic checking is unfinished |
| `src/utility/compiler.rs` | Public compiler input types and stage orchestration |
| `src/utility/error.rs` | Error categories and diagnostic formatting |
| `src/backend/` | Wrapper types around `an-inkwell` and LLVM objects |

The repository also contains `src/extra/`, including embedded-C and pointer/cast helpers, and `src/backend/shortcuts.rs` for small backend operations.

## Public compiler surface

The library exports `Source`, `Sources`, `Compiler`, `CompilerOptions`, `Target`, `Output`, `Error`, and several language representation types. A `Source` contains a name and text. `Sources::new` rejects duplicate source names. The currently defined target is `Target::Native`; output variants are `Output::Executable` and `Output::Object`.

These options are data structures, not proof that the output pipeline is implemented. `compile_with_options` validates the options and prepares each parsed semantic program, then returns `Error::StageNotImplemented(Stage::Semantic)`.

## Backend boundary

The backend uses `an-inkwell` rather than exposing LLVM's full object model throughout the compiler. The current wrappers include an LLVM context, modules, functions, and instruction handlers. Tests demonstrate construction of simple `i32` and `void` functions and return instructions. Full lowering from a semantically checked Astry program to LLVM IR and native output remains unfinished.

## Determinism and errors

Source units and declarations should be processed in stable order. Diagnostics and emitted symbol names should be deterministic. Errors are explicit; the design does not call for silently recovering from an unsupported target or failed stage.
