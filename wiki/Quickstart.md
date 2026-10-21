# Quick start

This page covers the repository's current development workflow. It does not imply that Astry can already compile a complete program to native output.

## Requirements

- Rust and Cargo with support for the crate's Rust 2024 edition.
- The LLVM/toolchain setup required by [`an-inkwell`](https://github.com/Censera/an-inkwell) version 26.
- Git.

The repository's `.cargo/config.toml` sets `LLVM_SYS_no_polly = "1"` and `LLVM_SYS_no_ffi_libs = "1"`. Consult `an-inkwell` and the installed LLVM package for the remaining build requirements.

## Get the source

```console
git clone https://github.com/Censera/astry.git
cd astry
```

The default branch reported by the repository is `entry`.

## Build and test the Rust project

```console
cargo build
cargo test
```

These commands build and test the compiler implementation. They do not demonstrate end-to-end native compilation of an Astry program.

## Current CLI

The current command-line entry point accepts a single source-file path:

```console
cargo run -- path/to/main.astry
```

For example, the language specification uses function declarations such as:

```rs
fn i32 add(a i32, b i32) {
    return a + b
}

fn i32 main() {
    return add(20, 22)
}
```

At this stage, a syntactically valid program reaches the semantic stage and the compiler returns an error equivalent to:

```text
compiler stage is not implemented: Semantic
```

The CLI does not yet expose output-path, target, optimization, check-only, or IR-emission options. Those are tracked in `todo/entry.md`.

## Explore the code

- `src/lexer/lexer.rs`: source tokenization.
- `src/parser/`: declarations, expressions, and AST construction.
- `src/semantic/`: semantic-analysis data structures and type syntax.
- `src/backend/`: the LLVM wrapper layer.
- `src/utility/compiler.rs`: compiler entry points and stage orchestration.
- `doc/syntax.md`: authoritative language syntax.

See [Compiler architecture](Compiler-Architecture.md) and [Status and roadmap](Status-and-Roadmap.md) for the current boundaries.
