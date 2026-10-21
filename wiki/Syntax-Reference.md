# Syntax reference

This page collects the rules most likely to affect everyday reading. The complete and authoritative reference is [`doc/syntax.md`](../doc/syntax.md), including examples, compiler notes, and open questions `Q1` through `Q20`.

## Tokens and terminators

- Most statements end with `;`.
- `mod` and `use` lines do not take a semicolon.
- A final expression in a block may omit `;`; that determines whether the block yields a value.
- `'x'` is a character literal. A form such as `'outer` is a loop label.
- `^` is used for pointer types and postfix dereference; `&` is prefix address-of.
- `->` is an explicit cast.
- `..` and `..=` form half-open and closed ranges.

The keyword set in v26 is: `mod use let const fn return if elif else loop while for in break continue match enum struct type into pub pri embed macro and or xor not true false None`.

## Declarations

```rs
mod game
use { standard, math }

let score i32 = 0
const MAX_SCORE i32 = 100

fn i32 add(a i32, b i32) {
    return a + b
}
```

The type follows a binding name and a function's return type precedes its name. Explicit types are optional for bindings but are part of the function form described in the syntax specification.

Visibility markers are `pub` and `pri`. The default visibility has not been decided in the specification.

## Block values

A block consists of items inside braces. A final expression without `;` becomes the block's value; ending that expression with `;` discards the value. A block with no final value is invalid in a context that requires a value.

```rs
let value = {
    let a = 20;
    let b = 22;
    a + b
}
```

## Calls and braces

A brace after a name can mean a struct literal, a call with block arguments, or a control-flow body. The parser uses the declared kind of the name to tell them apart. A function used as a condition must be called explicitly:

```rs
if check() { ... }
```

The syntax document explains why `if check { ... }` is parsed differently when `check` is known to be a function. Calls using parentheses are the unambiguous form.

## Operators

| Family | Forms |
|---|---|
| Logical | `&&`, `||`, `~~`, `!`; keywords `and`, `or`, `xor`, `not` |
| Bitwise | `:&`, `:|`, `:^`, `:<`, `:>`, `<:` |
| Comparison | `==`, `!=`, `<`, `>`, `<=`, `>=` |
| Range | `..`, `..=` |
| Pointer | `&value`, `value^` |
| Cast | `value -> type` |
| Assignment | `=`, `+=`, `-=`, `*=`, `/=` |

The design intentionally keeps logical and bitwise operators in different symbol families. Plain `&` is address-of, not bitwise and. The precedence and associativity rules are not fully settled in the v26 document.

## Collections

- Arrays use `T[n]` and array literals such as `[1, 2, 3]`.
- Vectors use `T<n>` and vector literals such as `<1, 2, 3>`.
- Tuples use `(a, b)` and support tuple binding forms.
- Ranges can be used by `for`; range slicing is not fully specified for every collection.

Vector indexing is written `vector<index>`, which depends on the parser knowing the name is a vector. In other contexts, `<` can mean comparison. Parenthesize comparisons inside an index.

## Functions and flags

`@Stripped` allows a function call to omit parentheses. `@Loosely` describes functions whose parameter list contains `...`. Both forms affect how the parser determines argument boundaries. Their complete rules and limitations are in [`doc/syntax.md`](../doc/syntax.md).

## Open questions

Do not treat items marked `Qn` in the source specification as settled language behavior. They include default visibility, import binding, multiple-binding semantics, uninitialized values, block return rules, conditional expressions without `else`, match wildcard/exhaustiveness, enum patterns, variadic arguments, method receivers, struct declaration equivalence, cast rules, operator precedence, macro substitution, pointer lifetimes and narrowing, heap ownership, nested unions, vector length/capacity, slicing, and numeric-literal defaults.
