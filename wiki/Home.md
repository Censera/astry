# Astry Wiki

Astry is a statically typed compiled language designed for game development. Its v26 design uses explicit syntax for types, pointers, casts, and logical versus bitwise operators. The compiler is written in Rust and uses [`an-inkwell`](https://github.com/Censera/an-inkwell) as its LLVM interface.

> **Implementation status:** Astry's specification describes more language behavior than the current compiler implements. The CLI reaches the semantic stage and then returns `StageNotImplemented(Semantic)`. End-to-end code generation is not available yet.

## Start here

- [Quick start](Quickstart.md): build the Rust project and understand the current CLI.
- [Language overview](Language-Overview.md): the main concepts and a small example.
- [Syntax reference](Syntax-Reference.md): operators, declarations, blocks, types, and grammar rules.
- [Compiler architecture](Compiler-Architecture.md): the stages and ownership boundaries.
- [Diagnostics](Diagnostics.md): error stages and current output.
- [Status and roadmap](Status-and-Roadmap.md): implemented groundwork, missing pieces, and unresolved language decisions.
- [Contributing](Contributing.md): how to change the implementation and keep the specification consistent.

## Source of truth

[`doc/syntax.md`](../doc/syntax.md) is the source of truth for v26 spelling and grammar. It also lists open questions marked `Q1` through `Q20`; those behaviors are not settled. The other design documents explain the purpose, architecture, and planned work, but do not override the syntax specification.

## Project boundaries

The intended pipeline is:

```text
source
  -> lexer
  -> return-type normalization
  -> macro expansion
  -> declaration index
  -> parser / AST
  -> semantic analysis
  -> LLVM IR
  -> native output
```

The early token and parser stages have implementation and tests. Semantic analysis and native output are not complete. See [Status and roadmap](Status-and-Roadmap.md) before relying on a feature described by the specification.
