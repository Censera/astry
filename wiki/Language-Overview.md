# Language overview

This page summarizes the v26 language design. It is a guide to the specification, not a promise that every listed feature is implemented in the compiler.

## A small example

```rs
fn i32 clamp(value i32, low i32, high i32) {
    if value < low {
        return low
    } elif value > high {
        return high
    } else {
        return value
    }
}

fn i32 main() {
    let value = clamp(12, 0, 10);
    return value
}
```

Function return types appear before the function name. Parameters use `name type`; a binding may infer its type or declare it explicitly. The semicolon on a block's final expression matters: an expression without `;` can be the block's value.

## Bindings

`let` declares a reassignable binding. `const` declares a binding that cannot be assigned again. A type annotation follows the name.

```rs
let count = 0;
let ratio f64 = 0.5;
const LIMIT u32 = 100;

count = count + 1;
```

Shadowing is allowed: a later declaration can hide an earlier binding in its scope. It creates a new binding rather than modifying the old one.

## Functions and calls

```rs
fn i32 add(a i32, b i32) {
    return a + b
}

let result = add(20, 22);
```

Functions can be overloaded. The design allows overloads to differ by parameter types/count or return type; when overloads differ only by return type, the expected type must disambiguate them.

The `@Stripped` flag permits calls without parentheses, and `@Loosely` marks a parameter list containing `...`. These forms have unusual argument-boundary rules. See `doc/syntax.md` before using them.

## Blocks and control flow

The specification includes `if`, `elif`, `else`, `loop`, `while`, `for`, `break`, `continue`, and `match`.

```rs
for i in 0..10 {
    if i == 5 { break }
}
```

`..` is a half-open range and `..=` is a closed range. Loops may have labels; a labeled `break` or `continue` targets that loop.

A block can produce a value when its last item is an expression without a trailing semicolon:

```rs
let sum = {
    let a = 20;
    let b = 22;
    a + b
}
```

Some block and control-flow details remain open in the syntax document, including what a conditional expression without `else` yields.

## Types

The v26 type list includes:

| Syntax | Meaning in the specification |
|---|---|
| `i8` through `i64` | Signed integers |
| `u8` through `u64` | Unsigned integers |
| `f32`, `f64` | Floating-point values |
| `bool`, `char`, `string` | Boolean, character, and string values |
| `^T` | Non-null pointer to `T` |
| `?^T` | Optional pointer to `T` |
| `T[n]` | Fixed-length array |
| `T<n>` | Growable vector |
| `(A, B)` | Tuple |
| `Union<A, B>` | Value holding one member type |
| Named types | Structs, enums, and aliases declared by the program |

The default type for unsuffixed numeric literals is unresolved. Memory ownership for strings and vectors is also unresolved.

## Pointers and casts

The design makes pointer operations explicit:

```rs
let value i32 = 5;
let pointer ^i32 = &value;
let copy i32 = pointer^;
let widened ?^i32 = pointer;
```

`&value` takes an address, postfix `^` dereferences a pointer, and `->` performs an explicit cast:

```rs
let digit = 1 -> char;
```

The distinction between `^T` and `?^T` is checked in semantic analysis. Lifetime behavior and some optional-pointer conversions remain open questions.

## Operators

Logical operators use `&&`, `||`, `~~`, and `!`, with keyword alternatives `and`, `or`, `xor`, and `not`. Bitwise operators use a separate family: `:&`, `:|`, `:^`, `:<`, `:>`, and `<:`. The specification rejects plain `&` as a bitwise operator.

The syntax document still lists operator precedence and some increment/decrement details as open questions. Parenthesize ambiguous expressions until the rules are settled.

## More language constructs

The syntax specification also describes modules and imports, structs, enums, `into` blocks, user-defined types, unions, macros, embedded C, and experimental lambdas. Several construction, dispatch, type, and lifetime rules remain unresolved. See [Syntax reference](Syntax-Reference.md) and the authoritative [`doc/syntax.md`](../doc/syntax.md).
