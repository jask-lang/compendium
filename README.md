# The jask Language — Compendium

A complete, source-derived reference for the **jask** programming language. Every chapter is built
directly from the interpreter's C# source (lexer, parser, and interpreter) so that syntax, semantics,
operator precedence, error messages, and permission behavior are documented as they actually behave —
including boundaries, edge cases, and quirks.

> **What is jask?** A dynamically-typed, imperative scripting language with first-class functions,
> struct types, lists, maps, modules, a permission + trust security model, and an embedded standard
> library (`jcore`). It is interpreted (no compilation step) and ships as a single executable.

---

## How to use this compendium

Start with [Getting Started](01-getting-started.md), then work through the language core in order.
The **Reference** chapters (built-in functions, standard library) are intended to be consulted
whenever you need a specific function. Every chapter links back to this index and to related chapters.

Source paths in this document are relative to the repository root:

| Piece | Source file(s) |
|-------|----------------|
| Tokenizer / literals / comments | `interpreter/Lexer.cs` |
| Grammar / statements / expressions / precedence | `interpreter/Parser.cs` |
| Runtime semantics / scoping / control flow | `interpreter/Interpreter.cs` |
| Expression evaluation / operators / calls | `interpreter/Interpreter/Interpreter.Expressions.cs` |
| Built-in functions | `interpreter/Interpreter/Interpreter.InternalFunctions*.cs` |
| Permissions | `interpreter/Permissions.cs` |
| Standard library | `jcore/math.jask`, `jcore/string.jask`, `jcore/date.jask` |

---

## Table of Contents

### Getting Started
1. [Getting Started](01-getting-started.md) — installation, running scripts, the CLI, permissions flags, the REPL.

### Language Core
2. [Language Basics](02-language-basics.md) — syntax, comments, identifiers, numbers, strings, booleans, `nil`, every operator, precedence, truthiness, and equality (`==` vs `is`).
3. [Variables and Scoping](03-variables-and-scoping.md) — `set`, local vs global scope, `set global`, field assignment, `restrict`, shadowing rules.
4. [Control Flow](04-control-flow.md) — `if`/`else if`/`else`, `while`, `for...in` (lists, maps, strings), `break`/`continue`, `return`.
5. [Data Structures](05-data-structures.md) — lists, maps, struct instances, bracket `[...]` and member `.field` access, iteration, and structural equality.
6. [Functions](06-functions.md) — definition, typed parameters, defaults, overloading, named arguments, return values, and call resolution.
7. [Structs](07-structs.md) — defining struct types, default field values, instantiation, field access, immutable field updates, and introspection.

### Robustness & Packaging
8. [Error Handling](08-error-handling.md) — the error model, `try`/`catch`/`endtry`, the `Error` object, `assert`.
9. [Modules](09-modules.md) — `use ... as`, the module system, embedded vs file-system modules, `export`, and module calls.
10. [Security and Permissions](10-security-and-permissions.md) — the permission model, CLI flags, path-based permissions, and the trust/verify/untrust system.

### Reference
11. [Built-in Functions](11-built-in-functions.md) — all **51** interpreter functions with exact signatures, returns, and error messages: core, string/char, list, map, I/O, HTTP, modules, struct, trust.
12. [jcore Standard Library](12-jcore-standard-library.md) — the `math` (8), `string` (29), and `date` (16 + the `Date` struct) modules, written in jask.

### Deep Dives
13. [Advanced Topics](13-advanced-topics.md) — `interpret()` (dynamic code), interpreter architecture, error attribution, limitations and known quirks, and testing conventions.

---

## Quick reference

### Reserved words

| | | | |
|---|---|---|---|
| `set` | `restrict` | `global` | `in` |
| `if` | `else` | `endif` | `while` |
| `endwhile` | `for` | `endfor` | `true` |
| `false` | `nil` | `function` | `endfunction` |
| `use` | `as` | `struct` | `endstruct` |
| `return` | `break` | `continue` | `and` |
| `or` | `not` | `is` | `try` |
| `catch` | `endtry` | `export` | |

> Identifiers must **start with a lowercase letter** (variables, functions, module aliases) or an
> underscore; **struct type names must start with an uppercase letter**. See
> [Language Basics](02-language-basics.md#identifiers-and-keywords).

### Operators

| Category | Operators | Notes |
|----------|-----------|-------|
| Arithmetic | `+` `-` `*` `/` `%` | `+` adds numbers **or** concatenates (see [Basics](02-language-basics.md#the-plus-operator)); `/ 0` is an error |
| Comparison | `==` `!=` `<` `>` `<=` `>=` | `==` is structural; `!=` its negation; `< > <= >=` are numeric only |
| Logical | `and` `or` `not` | short-circuit; `not` is a prefix unary |
| Identity | `is` | reference identity (contrast with `==`) |
| Access | `.` `[ ]` `::` | member access, index, module-qualified call |
| Grouping / call | `( )` | expression grouping and function calls |
| Assignment | `=` | only inside `set` / `set global` / function-parameter defaults |
| Unary | `-` | numeric negation |

### How values are displayed

`print`, `toString`, and the REPL all stringify identically:

| Value | Rendered |
|-------|----------|
| `nil` / `true` / `false` | `nil` / `true` / `false` |
| number | integral values drop the dot (`4.0` → `4`); else invariant (`4.5`) |
| string | raw, no quotes |
| list | `[1, "a"]` (string members quoted) |
| map | `["k" = v]` (string keys/values quoted) |
| struct | `Point { x: 1, y: 2.5 }` |
| untrusted | `untrusted` |

### Running jask

```bash
# interactive REPL
dotnet run

# run a script (grant the permissions it needs)
dotnet run --allow-stdout --input "myScript.jask"
```

Full details in [Getting Started](01-getting-started.md#running-jask).

---

*Generated from the interpreter source. If behavior ever disagrees with this document, the source
(`interpreter/`) is authoritative.*
