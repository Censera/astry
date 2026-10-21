# Astry

Astry is a statically typed compiled programming language designed for game development. The v26 design aims for explicit types, pointers, casts, and a compiler pipeline that remains understandable from source text to generated code.

**The language specification is ahead of the implementation.** The current compiler tokenizes and parses source, but `Compiler::compile_with_options` stops with `StageNotImplemented(Semantic)`. It does not yet produce an executable or object file end to end.

## Documentation

- [Wiki home](wiki/Home.md)
- [Quick start](wiki/Quickstart.md)
- [Language overview](wiki/Language-Overview.md)
- [Syntax reference](wiki/Syntax-Reference.md)
- [Compiler architecture](wiki/Compiler-Architecture.md)
- [Diagnostics](wiki/Diagnostics.md)
- [Status and roadmap](wiki/Status-and-Roadmap.md)
- [Contributing](wiki/Contributing.md)

The authoritative v26 grammar and examples live in [`doc/syntax.md`](doc/syntax.md). The design, goals, and planned implementation order are in [`doc/design.md`](doc/design.md), [`doc/goal.md`](doc/goal.md), and [`doc/plan.md`](doc/plan.md).
