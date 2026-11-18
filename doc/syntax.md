# Astry v26 Syntax

This file is the source of truth for Astry's spelling and grammar in v26. `goal.md` states what the language is for, `design.md` describes the compiler's shape, and `plan.md` gives the order of work.

Every section gives the rule and examples, followed where useful by notes for semantic analysis (SeMa) and the backend. Unresolved behavior is marked `(Qn)` and collected under "Open questions". Features deliberately left for later development are listed in "Postponed features" at the end.

## Reading guide

- `T` is a type placeholder, `name` is an identifier, and `value` is an expression.
- `// error` on an example line means the compiler rejects it.
- `print` in examples is a library function declared as `fn print(text string)`. It is not a language construct.

### Principles

- Types, pointers, and casts have explicit syntax.
- Logical and bitwise operators use separate keyword families.
- Blocks provide statement scope; ordinary blocks are not expressions and do not produce values.
- A function body may end with an expression without `;`; that expression is the function's return value.
- Function calls always use parentheses. Function declarations have one unambiguous form.
- The parser uses declaration information only where syntax requires it. Type checking stays in SeMa.
- Features not needed by the core are recorded under "Postponed features" and do not shape the current grammar.

### Compiler pipeline

```text
source text
  -> lexer                     tokens
  -> declaration index         declared names and kinds
  -> parser                    AST
  -> semantic analysis         resolved names, types, module graph
  -> backend                   LLVM IR
```

The declaration index scans tokens before parsing so type names and vector bindings are known where the grammar needs to distinguish them.

## 1. Lexical rules

### Tokens and keywords that need a note

| Token | Name | Meaning |
| --- | --- | --- |
| `^` | Caret | pointer type (prefix) and dereference (postfix). One token, so a pointer to a pointer is two carets. |
| `&` | | address of (prefix). |
| `&&` | And | logical and. `&&x` is never two address-ofs. Write `& &x`. |
| `~~` | Xor | logical xor. |
| `as` | | explicit cast. |
| `::` | Type scope | access to a member associated with a type. |
| `..` `..=` | | half-open and closed range. |
| `:and` `:or` `:xor` `:not` `:right` `:left` | | bitwise operator family. |

The colon-prefixed bitwise spellings are each single operator tokens. `<<` and `>>` do not exist. Nested angle brackets close without splitting a token: `v<w<index>>` lexes as `v < w < index > >`.

### Comments, literals, terminators

```rs
// line comment
let a = 10;          // integer
let b = 2.5;         // float
let c = 'x';         // char
let d = "text";      // string
let e = true;        // bool
```

- Declarations, assignments, and other expression statements end in `;`.
- `mod` and `use` lines take no `;`.
- A semicolon-terminated expression statement discards its value.
- The final expression in a function body may omit `;`; its value is returned by the function. This rule applies only to the function body's final expression. Ordinary nested blocks remain statement scopes and never produce values.

### Labels and characters

The lexer looks ahead to tell them apart. `'a'` closes on a second quote and is a char. `'a` doesn't and is a loop label.

```rs
let c = 'a';         // char
loop 'a { break 'a } // label
```

### Keywords

`mod use let const fn ext return if elif else loop while for in break continue match enum struct type apply pub pri as and or xor not true false None`

## 2. Modules and imports

`mod` names the source unit's root module. `use` brings names in from other modules. A unit has one root module, and module names are unique within a compilation.

```rs
mod mygame

use library
use { mygame, standard, memory, engine }
use math { function, variable }
```

- `use name` imports one module.
- `use { a, b, c }` imports several modules.
- `use math { x, y }` imports selected names from `math`.
- Selection is one level deep. Nested import paths are not part of the current grammar.

The driver reads every unit's imports before parsing so imported names are available to the declaration index. SeMa builds the module graph and rejects cycles and conflicts.

Visibility is `pub` or `pri` on items, struct fields, enum variants, and type-associated functions declared in `apply` blocks. The default when neither is written is undecided (Q1).

## 3. Bindings

`let` declares a binding that can be assigned again. `const` declares one that can't. The type follows the name and is optional.

```rs
let count = 0;               // type inferred
let ratio f64 = 0.5;         // type written
let x, y, z = 0;             // three bindings (Q2)
let _ = compute();           // evaluated, result dropped

const LIMIT u32 = 100;
const A, B = 1;
```

### Brace form

`{ ... }` after `let` or `const` groups declarations. The group itself has no value or shared initializer. Each entry is a normal binding declaration and may have its own type and initializer. Entries are processed in order, so an entry may use earlier entries.

```rs
let {
    width  i32 = 80,
    height i32 = 24,
    area   i32 = width * height
};

const {
    W = 80,
    H u32 = 24
};
```

```rs
let { a = b, b = 1 };        // error: b is declared later
```

### Assignment and const

```rs
let n = 1;
n = 2;                       // ok
LIMIT = 5;                   // error: LIMIT is const
```

### Shadowing

Shadowing is allowed. A later `let` of the same name creates a new binding and hides the old one from that point on. The old binding is not modified.

```rs
let n = 1;
let n = n + 1;               // new n, value 2
{
    let n = 10;              // inner n
}
// n is 2 here
```

## 4. Functions

A function is `fn`, a name, a parameter list, an optional return type, and a body. The return type follows the parameter list.

```rs
fn main() {}
fn five() i32 { 5 }
fn add(a i32, b i32) i32 { a + b }
fn maybe(p ^i32) ?^i32 { p }

fn choose(ok bool) i32 {
    if ok { return 1; }
    2
}
```

Parameters are `name T`, separated by commas. A function without a return type does not return a value. A function with a return type may end with a final expression without `;`; the expression's type must match the declared return type. If the final expression has `;`, its value is discarded and it does not satisfy the return type.

`return;` leaves the function immediately. `return value;` returns a value immediately and is useful for early exits.

A function name is declared once in its containing namespace. Declaring the same function name again is an error, regardless of its parameter or return types. Function overloading, including return-type overloading, is not supported. Calls always use parentheses.

### Type-associated functions

`apply Type { ... }` declares functions associated with a type. They are accessed with `Type::function(...)`. The `::` operator is described under Types.

```rs
type Node struct { value i32 }

apply Node {
    pub fn new() Node { Node { value = 0 } }
    pri fn secret() i32 { 1 }
}

let node Node = Node::new();
```

How a function associated with a type receives an instance value is not defined in the current core (Q5).

## 5. Control flow

### if, elif, else

```rs
if n < 0 {
    return -1
} elif n == 0 {
    return 0
} else {
    return 1
}

if done { break }
if skip { continue } else { break }
if check() { ... }
```

Conditions are expressions. Function calls in conditions use parentheses.

### loop, while, for

```rs
loop {
    if done { break }
}

while i < 10 { i += 1 }

for item in items {}
for i in 0..10 {}            // 0, 1, ... 9
for i in 0..=9 {}            // 0, 1, ... 9
```

### Labels

`loop 'name` names a loop. `break 'name` and `continue 'name` act on the named loop.

```rs
loop 'outer {
    loop {
        if found { break 'outer }
        if skip { continue }
    }
    loop {
        if finished { break 'outer }
    }
}
```

### Ranges

Ranges are half-open with `..` and closed with `..=`. They are supported by `for` loops. Range slicing is postponed.

```rs
for i in 0..10 {}
for i in 0..=9 {}
```

### match

`match` takes a value and a list of arms. An arm is a pattern and a statement block, separated by commas.

```rs
match n {
    0 { print("zero") },
    1 { print("one") },
    _ { print("many") }
}
```

Wildcard behavior and exhaustiveness rules are undecided (Q4).

## 6. Blocks, scope, and assignment

A block is a sequence of statements enclosed in braces. It creates a lexical scope and never produces a value.

```text
block = "{" statement* "}"
```

Statements inside a block execute in order. A nested block creates a nested scope.

```rs
fn main() {
    let outer i32 = 10;

    {
        let inner i32 = outer * 2;
        print("inside scope");
    }

    // inner is not visible here
}
```

### Scope

Bindings are visible from their declaration to the end of the block. Shadowing is allowed: a later `let` of the same name creates a new binding and hides the earlier one from that point on.

```rs
let n = 1;
let n = n + 1;               // new n, value 2
{
    let n = 10;              // inner n
}
// n is 2 here
```

### Assignment

Assignment evaluates the right side, stores it in the assignable left side, and produces the stored value. The left side must be assignable: a mutable binding, an element, a field, or a dereference. Whether a dereferenced pointer is assignable is undecided (Q9).

```rs
let a i32 = 123;
let b i32 = 456;
let t i32 = a;

a = b;                       // statement
let z = (a = b);              // z receives the assigned value
```

Declaration and assignment stay separate:

```rs
let value i32 = 10;
value = 20;
value i32 = 20;              // error: assignment doesn't declare
```

Function arguments are expressions enclosed in parentheses. Blocks cannot be passed as values or arguments.

## 7. Types

| Type | What it is |
| --- | --- |
| `i8` `i16` `i32` `i64` | signed integers |
| `u8` `u16` `u32` `u64` | unsigned integers |
| `f32` `f64` | floating point |
| `bool` | `true` or `false` |
| `char` | character |
| `string` | heap-allocated string |
| `^T` | non-null pointer to `T` |
| `?^T` | pointer to `T`, or `None` |
| `T[n]` | fixed-length array |
| `T<n>` | growable vector |

The type of an unsuffixed integer or float literal is undecided (Q12).

### Type-scope access

`Type::member` accesses a member associated with a type. It applies to built-in types, structs, declared types, and enums. Use `.` for a member accessed through a value; use `::` when the left side is a type name.

```rs
u8::max
i32::min
Point::origin()
Color::red
```

Built-in numeric limits are associated constants, `apply` declares type-associated functions, and enum variants are accessed through their enum type. `::` is one general operator, not syntax reserved for numeric constants.

## 8. Pointers

Two type spellings and two operators cover the whole feature:

| Syntax | Reads as | Where |
| --- | --- | --- |
| `^T` | non-null pointer to `T` | type position |
| `?^T` | pointer to `T`, or `None` | type position |
| `&v` | the address of `v` | expression, prefix |
| `v^` | the value `v` points to | expression, postfix |

```rs
let x i32 = 5;
let p ^i32 = &x;             // p holds the address of x
let y i32 = p^;              // y is 5
```

`^T` is never null. `?^T` can be `None`. A `^T` converts to `?^T`. The reverse is an error.

```rs
let a ^i32 = &x;
let b ?^i32 = &x;            // ok: ^i32 widens to ?^i32
let c ?^i32 = None;          // ok
let d ^i32 = None;           // error: ^i32 is never None
let e ^i32 = c;              // error: c may be None
let f i32 = c^;              // error: c may be None (narrowing: Q9)
```

### Pointer to pointer

Two carets, no `^^` token. Dereference chains the same way.

```rs
let x i32 = 5;
let p ^i32 = &x;
let pp ^^i32 = & &p;         // & & because && is logical and
let y i32 = pp^^;            // 5
```

### Through a pointer

`v^` can be read. Whether it can be assigned to is undecided (Q9).

```rs
fn add_one(p ^i32) {
    p^ = p^ + 1;
}

let n i32 = 1;
add_one(&n);                 // n is 2
```

### Address-of needs an addressable operand

```rs
let p ^i32 = &x;             // binding
let q ^i32 = &arr[2];        // element
let r ^f64 = &point.x;       // field
let s ^i32 = &5;             // error: a literal has no address
```

### Pointers and members

Postfix operators apply left to right. `p^.x` dereferences, then reads a field.

```rs
let p ^Point = &pt;
let x = p^.x;
```

### Compiler notes

- SeMa keeps `^T` and `?^T` as distinct types and checks the conversions above.
- The backend lowers both to LLVM `ptr`. Nullability exists only in SeMa.
- `&x` forces `x` into a stack slot (an `alloca`). `v^` is a load and `v^ = e` is a store. The load type comes from the pointee type in SeMa, because LLVM pointers are opaque.
- Returning `&local` from a function leaves a dangling pointer. No rule covers it yet (Q9).

## 9. Arrays

Arrays have a fixed length written in the type.

```rs
let a i32[3];                        // three elements
let b i32[] = [1, 2, 3];             // length from the initializer
let c i32[3] = [1, 2, 3];            // length checked against the initializer
let d i32[8, 0];                     // eight elements, all 0

let first = b[0];
let picked = b[a[1]];                // index with an index
b[0] = 9;                            // assign to an element
```

```rs
let e i32[2] = [1, 2, 3];            // error: three values for length two
```

The contents of `a` with no initializer are undecided (Q3). Range slicing is postponed.

## 10. Vectors

Vectors are growable and heap-backed. The element type is written before `<...>`.

```rs
let v i32<>  = <1, 2, 3>;            // length from the initializer
let w i32<3> = <1, 2, 3>;            // length checked
let x i32<8, 0>;                     // eight elements, all 0
let y i32<4>;                        // length 4 or capacity 4: Q11

let first = v<0>;                    // index
let mixed = v<w<0>>;                 // index with an index
v<1> = 20;                           // assign to an element

v.push(4);
v.pop();
let n = v.length();
```

`name<index>` is an index when `name` is a vector and a comparison otherwise. The parser decides from the name's kind.

```rs
let i = 3;
let ok = i < 10;                     // i is a value: comparison
let e = v<i + 1>;                    // v is a vector: index
```

Inside an index, `>` ends the index. A comparison there needs parentheses.

```rs
let bad = v<a > b>;                  // the first > ends the index
let fine = v<(a > b)>;
```

### Compiler notes

- SeMa types a vector as `Vector(T)`. `v<i>` needs an integer index and yields `T`. `push(x)` needs `x` convertible to `T`. Literal elements must agree on one `T`.
- `push`, `pop`, and `length` are vector operations. Their instance-receiver and dispatch rules are not defined in the current core (Q5).
- The backend emits `{ptr, len, cap}`. `push` reallocates when `len == cap`. Who frees the buffer is undecided (Q10).



## 11. Structs, enums, and user types

### Structs

```rs
pub struct Name {}
pri struct Name {}

struct Point {
    pub x f64,
    pub y f64,
    pri id i32,
}

type Point struct { x f64, y f64 }
```

The relationship between `struct Name {}` and `type Name struct {}` is undecided (Q6).

Literals are positional or named:

```rs
let a Point = Point { 3.0, 4.0 };
let b Point = Point { x = 3.0, y = 4.0 };
let c = Point { 1.0, 2.0 };
let d = Point { x = 1.0, y = 3.0 };
```

Member access:

```rs
let px = a.x;
a.y = 9.0;
```

### Enums

```rs
pub enum Name {}
pri enum Name {}

enum Shape {
    pub point,                       // no data
    pri circle(f64),                 // one value
    empty(),                         // empty list
    pub rect { w f64, h f64 },       // named fields
    hidden { id i32 }
}
```

Constructing and matching variants is undecided (Q7).

### apply blocks

`apply Name { ... }` declares functions associated with `Name`. They are accessed through `Name::function(...)`. Instance-receiver behavior is not part of the current core.

```rs
type Node struct {}

apply Node {
    fn new() Node { Node {} }
    pub fn depth() i32 { 0 }
}

let node Node = Node::new();
```

Enum variants are accessed through the enum type, for example `Shape::point`. The complete construction and matching rules remain open (Q7).

## 12. Casts

`value as type` converts explicitly. It is a postfix operator, so it binds like other postfix operators and chains from left to right.

```rs
let a = true;
let b = a as i8;             // 1
let c = b as char;           // '1'
let d = c as string;         // "1"
let e = d as char;           // '1'
let f = e as i8;             // 1
let g = f as bool;           // true

let n = true as i8 as char;  // '1'
```

Postfix means the cast binds tighter than arithmetic. Parenthesize the left side to cast a whole expression.

```rs
let a = x + y as i64;        // x + (y as i64)
let b = (x + y) as i64;      // cast the sum
```

Some conversions change representation, not only width. `1 as char` is `'1'`, the digit, and `'1' as i8` is `1`. Each pair of types has its own rule, so SeMa validates casts pair by pair. Pairs the draft doesn't list, and `n as char` for `n` outside 0..9, are undecided (Q8). Numeric conversions are explicit: the compiler does not silently widen an integer because its value would fit in a larger type. For example, `u8::max` has type `u8`; use `u8::max as u32` when a `u32` value is required.

### Combined example

This example uses explicit casts, type-associated numeric limits, bitwise operations, grouped declarations, and a final expression as the function result.

```rs
fn shade_color(base_color u32, dist f64, magnitude f64) u32 {
    let {
        shade  f64,
        red    u32,
        green  u32,
        blue   u32,
        alpha  u32,
        color  u32
    };

    shade = 1.0 / (1.0 + dist * magnitude);

    red = ((((base_color :right 24) :and (u8::max as u32)) as u8) as f64 * shade) as u32;
    green = ((((base_color :right 16) :and (u8::max as u32)) as u8) as f64 * shade) as u32;
    blue = ((((base_color :right 8) :and (u8::max as u32)) as u8) as f64 * shade) as u32;

    color = (red :left 24) :or (green :left 16) :or (blue :left 8);
    alpha = base_color :and (u8::max as u32);

    color :or alpha
}
```

## 13. Operators

### Logical

| Symbol | Keyword | Meaning |
| --- | --- | --- |
| `&&` | `and` | and |
| `\|\|` | `or` | or |
| `~~` | `xor` | xor |
| `!` | `not` | not |

```rs
let ok = a > 0 && b > 0;
let ok2 = a > 0 and b > 0;
let either = x || y;
let differ = x ~~ y;
let flip = !done;
let flip2 = not done;
if not done { ... }
```

### Bitwise

| Operator | Meaning |
| --- | --- |
| `:and` | and |
| `:or` | or |
| `:xor` | xor |
| `:not` | not |
| `:right` | shift right |
| `:left` | shift left |

```rs
let masked = flags :and 255;
let both = x :or y;
let toggled = x :xor y;
let inverted = :not x;
let up = x :left 3;
let down = x :right 3;
```

The two families never mix. `a && b` is logical and, `a :and b` is bitwise and, and `a & b` is an error. `:and`, `:or`, and `:xor` require operands of the same integer type. The compiler does not insert an implicit widening conversion. Cast explicitly when the types differ, for example `value :and (u8::max as u32)`.

### Arithmetic, assignment, comparison

```rs
let s = a + b - c * d / e;
n++;                         // increment
n--;                         // decrement
total += 5;
total -= 1;
total *= 2;
total /= 4;
a == b; a != b; a > b; a < b; a >= b; a <= b;
```

Precedence, associativity, and prefix versus postfix `++`/`--` are undecided (Q13). Parenthesize until they are.



## 14. Foreign functions

`ext fn ... in <language>` declares a function whose implementation is supplied in another language. The function signature uses Astry types; the body uses the selected foreign language.

### C

```rs
ext fn add(a i32, b i32) i32 in C {
    return a + b;
}
```

The body is a C function body, not Astry. The compiler generates the surrounding C function declaration and definition from the Astry signature, then compiles the result as C. The compiler must validate or reject signatures whose Astry types have no defined C representation.

### Assembly

```rs
ext fn add(a i32, b i32) i32 in asm {
    ; x86-64 System V example
    mov eax, edi
    add eax, esi
    ret
}
```

The body is assembly, not Astry. The compiler passes it to the assembler for the compilation target. Register names, instructions, and other details are target-specific.

### External implementations

The body may be omitted when the implementation is supplied by a separately compiled object file or linked library:

```rs
ext fn add(a i32, b i32) i32 in C;
ext fn multiply(a i32, b i32) i32 in asm;
```

The build configuration supplies the required object files and libraries. SeMa checks calls against the Astry signature. The backend and build system must ensure that the signature agrees with the foreign implementation's ABI, including argument and return representation, calling convention, symbol naming, and required register or stack rules.

The foreign body is read as raw source for its declared language. It is not parsed as Astry, and Astry expressions cannot appear inside it.

## 15. Parsing with name kinds

The parser uses declaration information where the same token sequence can have different meanings. The main remaining distinction is between a type name, a vector binding, and an ordinary value.

```rs
Point { x = 3.0, y = 4.0 }   // struct literal: Point is a type
if flag { ... }              // body: flag is a value
let item = values<0>;        // vector index
let less = value < 3;        // comparison
```

| Kind | Declared by | Parser use |
| --- | --- | --- |
| Type | `struct`, `enum`, `type`, primitive types | Recognizes struct literals and `Type::member` access. |
| Vector | A binding or parameter typed `T<...>`, or initialized with `<...>` | Distinguishes `name<index>` from comparison. |
| Value | Any other binding, parameter, loop variable, or pattern variable | Ordinary expression name. |
| Unknown | Not declared anywhere the parser can see | Ordinary expression name; braces belong to the surrounding construct. |

### Rules

- Function calls always use parentheses: `name(arg)`. A `{` after a function name never starts call arguments.
- A `{` after a type name can begin a struct literal.
- After `if`, `elif`, `while`, `for ... in`, `match`, and `apply`, `{` opens the construct's body.
- Local names shadow outer names from their declaration onward, following "Blocks, scope, and assignment".
- Name resolution, visibility, and type checking stay in SeMa. The parser's scope stack tracks only the distinctions required by the grammar.
- `name<index>` is an index when `name` is a Vector and a comparison otherwise.

```rs
struct Point { x f64, y f64 }
let flag = true;
let v i32<> = <1, 2, 3>;

let p = Point { x = 1.0, y = 2.0 };   // Type: struct literal
if flag { ... }                       // Value: body condition
let a = v<0>;                         // Vector: index
let b = flag < 3;                     // Value: comparison
```

### Boundaries

The parser learns the kind of a name. It doesn't learn which declaration the name refers to, or its type. Name resolution and type checking stay in SeMa.

### Building the index

1. Split each unit's token stream into top-level items: kind, token range, and modifiers.
2. Read declared names. `struct`, `enum`, and `type` give a Type. `fn` records a function name and signature for SeMa. `let` and `const` give a Value, or a Vector when the type is `T<...>` or the initializer starts with `<`. `apply Name { ... }` records functions associated with `Name`; enum declarations record their variants.
3. Store a `NameIndex` for types, values, vectors, functions, enum variants, and type-associated members. Use ordered maps so diagnostics are deterministic.

The index never reports errors. When scanning an item fails, it records nothing, skips to the next top-level item, and lets the parser report the real error.

Compilation makes two passes over the ordered sources. The first lexes and indexes every unit. The second parses every unit with its visible names: its own index plus what its `use` lines select from other units.



## 16. Diagnostics

Every diagnostic carries a source name, a line, and a column. In the compiler that is `Error::Parse { line, column, message }`, with `with_source` adding the file name. Stages report in source order so output is the same on every run.

```text
Finished [bin] in X ms
Checked, and everything is OK.
W [src/file.as][Line][Column] Warning message
E [src/file.as][Line][Column] Error message
```

## Open questions

- Q1. Default visibility when neither `pub` nor `pri` is written, for items, fields, variants, and `apply` methods.
- Q2. In `let x, y, z = 0;`, does each name receive a copy of the value?
- Q3. Scalar declarations without an initializer (`let x i32;`), and the contents of `T[n]` and `T<n>` without one.
- Q4. `match` wildcard behavior and exhaustiveness rules.
- Q5. Whether type-associated functions can receive an instance value, and what receiver syntax would express that. The current core defines only type-associated functions accessed through `Type::member`.
- Q6. Whether `struct Name {}` and `type Name struct {}` mean the same thing.
- Q7. Enum variant construction syntax and how variants integrate with `match` patterns.
- Q8. Casts between type pairs not listed, and `n as char` when `n` is outside 0..9.
- Q9. Pointers: narrowing `?^T` to `^T`, assignability through `v^`, and the lifetime of `&local`.
- Q10. Who frees a vector's buffer and a `string`'s heap storage.
- Q11. Whether `T<n>` without values specifies an initial length or capacity, and the return types of `pop` and `length`.
- Q12. The default type of an unsuffixed integer or float literal.
- Q13. Operator precedence, associativity, and prefix versus postfix `++`/`--`.

## Explicitly excluded from the core

The current core has no macro system, function overloading, return-type overloading, block expressions, or block arguments. These constructs are not alternate spellings of existing features.

## Postponed features

| Feature | Status | Deferred scope |
| --- | --- | --- |
| Unions | Postponed | Union types, implicit member conversions, tagged storage, and matching by member type. |
| Tuples | Postponed | Tuple types, tuple values, destructuring, positional extraction, and skipped tuple slots. |
| Lambdas | Postponed | Anonymous functions and closures are outside the current core. |
| Shortened conditionals | Postponed | Forms such as `return x if condition`; use ordinary `if` blocks instead. |
| Range slicing | Postponed | Slicing arrays, vectors, and strings with `..` or `..=`; ranges remain available in `for` loops. |
| Nested import paths | Simplified in the core | Current imports select names one level deep. Deeper nested selections are deferred. |
