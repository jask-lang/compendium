# Language Basics

[← Back to the Compendium index](README.md) · Prev: [Getting Started](01-getting-started.md) · Next: [Variables and Scoping](03-variables-and-scoping.md)

This chapter covers the surface syntax of jask: how programs are structured, comments, identifiers,
literals, and every operator. The source of truth is [`Lexer.cs`](../interpreter/Lexer.cs) and
[`Parser.cs`](../interpreter/Parser.cs).

---

## Program structure

A jask program is a sequence of **statements**, one per line by convention.

- The lexer tracks line numbers but emits **no newline token**; statements are delimited by the
  expression grammar. As a rule, keep **one statement per line**.
- Whitespace (spaces, tabs, carriage returns) is ignored.
- There is **no** statement-terminating semicolon.

### Comments

| Form | Syntax | Example |
|------|--------|---------|
| Single-line | `;` to end of line | `; this is a comment` |
| Multi-line | `;;` … `;;` | `;; everything in between ;;` is ignored (line numbers are still tracked) |

```jask
set a = 1        ; single-line comment
;;
   a multi-line
   comment block
;;
```

---

## Identifiers and keywords

- Identifiers start with a **letter** or **`_`**, then letters, digits, or `_`
  (e.g. `x`, `user2`, `_temp`).
- **31 words are reserved** as keywords (listed in the [index](README.md#reserved-words)). They are
  matched **case-sensitively**, so `Set` is an identifier, not the `set` keyword.
- Naming rules enforced at parse time (only the **first character** is checked):
  - **variables, functions, parameters, module aliases** must start with a **lowercase** letter
    (`Variable 'Foo' must start with a lowercase letter`);
  - **struct type names** must start with an **uppercase** letter (`Struct 'foo' must start with an uppercase letter`).

An unknown character is a lexer error: `Unknown character '{c}'.`. A bare `!` (not part of `!=`) is
`Unexpected character '!'.`.

---

## Literals

### Numbers

Numbers are IEEE-754 **double-precision** floating point. There is a single numeric type
(`type(1)` → `"number"`).

- Integer or decimal: `42`, `3.14`. Parsed with invariant-culture rules.
- **No** scientific notation, **no** hex/binary/octal, **no** `+`/`-` sign inside the literal —
  a sign is a separate operator: `-5` is `-(5)`.
- Leading zeros are allowed: `007` → `7`.
- A `.` is only consumed as a decimal point when **followed by a digit**. So `1.` lexes as the number
  `1` followed by a `.` (member-access) token, and `1..2` lexes as `1`, `.`, `.`, `2`.

```jask
type(42)      ; "number"
type(3.14)    ; "number"
007           ; 7
```

### Strings

Strings are double-quoted. They may span multiple lines (a raw newline inside a string is allowed and
counts as a line break).

Escape sequences:

| Sequence | Meaning |
|----------|---------|
| `\n` | newline |
| `\t` | tab |
| `\r` | carriage return |
| `\\` | backslash |
| `\"` | double quote |

Any other escape (`\x`) is left as-is (the backslash is **kept** literally, e.g. `"\a"` is the two
characters `\a`). An unterminated string is `Unclosed string.` — this also fires when a string ends
with a backslash right before end-of-input (a trailing `\` consumes nothing it can escape).

```jask
set greeting = "hello"
set quoted   = "she said \"hi\""
set path     = "C:\\temp\\file.txt"
set multi    = "line one\nline two"
```

String concatenation is covered in [The `+` operator](#the-plus-operator).

### Booleans and nil

| Literal | Type | Notes |
|---------|------|-------|
| `true`, `false` | `"boolean"` | the only boolean values |
| `nil` | `"nil"` | the absence of a value (like `null`) |

```jask
type(true)  ; "boolean"
type(nil)   ; "nil"
```

> You cannot pass `nil` to a function or use it as a map key/struct value — see
> [Functions](06-functions.md#passing-arguments) and [Data Structures](05-data-structures.md#maps).

### How values are displayed (Stringify)

`print`/`printLine`, `toString`, the REPL result echo, and error messages all stringify values with the
same rules (source: [`Interpreter.HelperFunctions.cs`](../interpreter/Interpreter/Interpreter.HelperFunctions.cs)):

| Value | Rendered as |
|-------|-------------|
| `nil` | `nil` |
| boolean | `true` / `false` |
| number | integral values in `long` range drop the dot (`4.0` → `4`); otherwise invariant-culture `4.5`, `1.5e3`, … |
| string | the raw characters (**no** surrounding quotes) |
| list | `[a, "b", c]` — **string elements** are wrapped in double quotes, others stringified; empty → `[]` |
| map | `["k" = v, …]` — string keys/values quoted, ` = ` separator; empty → `[]` |
| struct | `TypeName { field: v, … }` — e.g. `Point { x: 1, y: 2.5 }` (space after `{`, `key: value`) |
| `MapEntry` | `MapEntry { key: "a", value: 1 }` |
| untrusted | `untrusted` (regardless of the wrapped value) |

```jask
toString(4.0)        ; "4"
toString([1, "a"])   ; '[1, "a"]'
toString({ "k" = 2 }) ; '["k" = 2]'
toString(Point(x = 1)) ; "Point { x: 1 }"
```

> Because list/map display embeds quotes around string members but not around the whole collection,
> the rendered form of a list/map is **not** valid jask source (you cannot `eval` it back without the
> matching built-ins).

---

## Expressions

An **expression** produces a value. Every statement is either a keyword statement (`set`, `if`,
`function`, …) or an **expression statement** (a bare expression, typically a function call):

```jask
1 + 1                ; expression statement — evaluated, result discarded (in a script)
printLine("hi")      ; a call expression statement
set x = 1 + 1        ; keyword statement
```

In the **REPL** the result of a top-level expression statement is printed automatically; in a script
it is discarded. See [Grouping and calls](#operator-precedence) for the grammar.

---

## Tokens and the `=` sign

`=` is **not** an operator. It only appears in a few specific syntaxes — a bare `x = 1` statement is a
syntax error (`set` is required). The places `=` is legal:

- `set x = …` / `set global x = …` / `set a.b = …` (assignments),
- function-parameter **defaults** (`function f(a: number = 0)`),
- **map literals** (`{ "k" = v }`),
- **named-argument** calls (`f(a = 1)`).

Multi-character operators are lexed greedily: `==`, `!=`, `<=`, `>=`, `::`. There are **no** compound
assignment tokens (`+=`, `++`, `->`, `&&`, `||`, `=>` do not exist) — logical operators are the
keywords `and` / `or`. A bare `!` (not part of `!=`) is the lexer error `Unexpected character '!'.`.

---

## Operators

### Arithmetic

| Operator | Meaning | Operand types |
|----------|---------|---------------|
| `+` | addition **or** concatenation | numbers → sum; otherwise → string concat (see below) |
| `-` | subtraction | numbers |
| `*` | multiplication | numbers |
| `/` | division | numbers — `/ 0` throws `Division by zero` |
| `%` | modulo (remainder) | numbers — same precedence as `*` and `/` |
| `-` (unary) | negation | numbers |

All non-`+` arithmetic operators require numeric operands; otherwise:
`Operator '-' expects a number, but received '...'` (the offending value is shown stringified, so a
string appears **unquoted**).

Modulo (`%`) uses IEEE-754 remainder semantics: the **sign of the result follows the dividend**
(`-7 % 2` → `-1`, `7 % -2` → `1`), it works on fractions (`7.5 % 2` → `1.5`), and `0.0 % 0.0` yields
`NaN` rather than an error (unlike `/ 0`).

### The plus operator (`+`)

`+` is the one polymorphic operator:

- if **both** operands are numbers → numeric addition;
- **otherwise** → `Stringify(left) + Stringify(right)` (string concatenation).

```jask
2 + 3        ; 5
"a" + "b"    ; "ab"
"a" + 1      ; "a1"    (1 is stringified)
"5" + "5"    ; "55"    (string concat, NOT 10)
1.5 + 2.5    ; 4.0
```

> Because of this, `"5" + 5` yields `"55"`, not `10`. Convert with `toNumber(...)` when you mean
> arithmetic. See [Built-in Functions](11-built-in-functions.md#core-functions).

### Comparison

`<`, `>`, `<=`, `>=` are **numeric-only** and return a boolean:

```jask
3 < 5        ; true
10 >= 10     ; true
```

Comparing strings or other types with these operators is an error
(`Operator '<' expects a number, but received '...'`).

### Equality and identity

| Operator | Meaning |
|----------|---------|
| `==` | **structural** (deep) value equality |
| `!=` | negation of `==` |
| `is` | **reference** identity (same object in memory) |

`==` compares by value, recursively:

```jask
1 == 1                ; true
"ab" == "ab"          ; true
[1, 2] == [1, 2]      ; true     (lists compare deeply)
1 == 1.0              ; true     (same number value)
nil == nil            ; true
```

Different **types** are never `==` (`1 == true` → `false`). **Lists** and **structs** compare
deeply (element / field by field). **Maps are the exception**: two distinct maps are `==`-equal
only if they are the *same* map object (see
[Data Structures → Equality](05-data-structures.md#equality)).

`is` is reference identity — two separately-created but equal values are **not** `is`:

```jask
set a = [1, 2]
set b = [1, 2]
a == b   ; true
a is b   ; false    (two distinct list objects)
a is a   ; true
```

> `is` is most useful for checking whether two names refer to the **same** struct/list/map instance
> (relevant to jask's immutable structs — see [Structs](07-structs.md#field-assignment-is-a-value-update)).

### Logical

| Operator | Meaning |
|----------|---------|
| `and` | logical AND, short-circuits, returns a boolean |
| `or`  | logical OR, short-circuits, returns a boolean |
| `not` | logical NOT (prefix), right-associative |

```jask
true and false   ; false
false and <anything> ; false (right side not evaluated)
true  or <anything> ; true  (right side not evaluated)
not true          ; false
not not false     ; true
```

Because `and`/`or` evaluate their operands and coerce with truthiness, they always yield a real
`boolean`. See [Truthiness](#truthiness).

---

## Operator precedence

jask parses expressions with a precedence-climbing grammar. From **tightest** (binds first) to
**loosest**:

| Rank | Operators | Associativity | Notes |
|------|-----------|---------------|-------|
| 1 (tightest) | Primary atoms: literals, `( … )`, `[ … ]` lists, `{ … }` maps, variables | — | |
| 2 | Postfix: call `(…)`, member `.f`, index `[…]`, module `mod::f(…)` | chains | `f(x).y[0]`, `a.b[c].d` |
| 3 | Unary minus `-` (negation) | right | binds **tighter** than `*`: `2 * -3` → `2 * (-3)` |
| 4 | `*` `/` `%` | left | all three share one precedence |
| 5 | `+` `-` (binary) | left | |
| 6 | `<` `>` `<=` `>=` | left | numeric only |
| 7 | `==` `!=` `is` | left | all three share one precedence |
| 8 | `not` (prefix) | right | binds **looser** than equality: `not x == y` → `not (x == y)` |
| 9 | `and` | left | short-circuits |
| 10 (loosest) | `or` | left | short-circuits |

Two non-obvious consequences worth memorizing:

- **`not` binds looser** than comparison, equality, and arithmetic, so it applies to a whole
  expression: `not 1 == 2` means `not (1 == 2)`. Because of this, `not` cannot be a *direct* operand of
  an arithmetic or comparison operator — write `a + (not b)` rather than `a + not b`.
- **Unary `-` binds tighter** than `*`/`/`/`%` but **looser** than the postfix level, so `2 * -3` is
  `2 * (-3)` and `-a.b` is `-(a.b)`.

Worked examples:

```jask
1 + 2 * 3          ; 7        (* binds tighter than +)
2 * -3             ; -6       (unary - binds tighter than *)
(1 + 2) * 3        ; 9        (parentheses override precedence)
2 + 3 * 4 / 2      ; 8        (* and / are equal and left-associative: 2 + (12 / 2))
not 1 == 2         ; true     (== is tighter than not → not (1 == 2))
not (1 < 2)        ; false    (not (true))
1 < 2 and 3 > 2    ; true     ((1 < 2) and (3 > 2))
a == b and c       ; (a == b) and c   (== is tighter than and)
```

Use parentheses to override precedence whenever in doubt.

---

## Truthiness

`if`, `while`, `and`, and `or` treat values as follows ([`IsTruthy`](../interpreter/Interpreter/Interpreter.Expressions.cs)):

| Value | Truthy? |
|-------|---------|
| `nil` | **false** |
| `false` | false |
| `true` | true |
| any number (including `0`) | **true** |
| any string (including `""`) | **true** |
| any list / map / struct (including `[]`) | **true** |

> Only `nil` and `false` are falsy. So `if 0` and `if ""` are **true** — unlike many languages.

```jask
if 0
    printLine("0 is truthy")
endif

while nil
    ; never runs
endwhile
```

---

## Next

- [Variables and Scoping](03-variables-and-scoping.md) — `set` and where variables live.
- [Control Flow](04-control-flow.md) — branches and loops.
- [Built-in Functions](11-built-in-functions.md) — `type`, `toNumber`, `toString`, and friends.
