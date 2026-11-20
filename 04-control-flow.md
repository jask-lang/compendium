# Control Flow

[← Back to the Compendium index](README.md) · Prev: [Variables and Scoping](03-variables-and-scoping.md) · Next: [Data Structures](05-data-structures.md)

jask uses **keyword-delimited** blocks (no braces): `if … endif`, `while … endwhile`, `for … endfor`,
`function … endfunction`, `struct … endstruct`, `try … endtry`. Blocks are indented by convention
(the indentation is not significant). All conditions use [truthiness](02-language-basics.md#truthiness).

Source of truth: [`Parser.cs`](../interpreter/Parser.cs) and the `Execute` switch in
[`Interpreter.cs`](../interpreter/Interpreter.cs).

---

## `if` / `else if` / `else`

```jask
set x = 10

if x > 5
    printLine("big")
else if x == 5
    printLine("five")
else
    printLine("small")
endif
```

- The condition is checked with truthiness; the **first** true branch runs.
- `else if` may be chained any number of times; a final `else` (with no `if`) is optional.
- An `else if` is distinguished from `else` by the `if` keyword that follows it.
- There is no `else if` *without* the leading `if` — the form is `else if …`.

Minimal forms:

```jask
if x > 0
    printLine("positive")
endif

; single-line bodies are allowed
if x > 0 printLine("positive") endif
```

> The condition does **not** need parentheses or to be a boolean: `if 0` runs (0 is truthy),
> `if ""` runs, `if nil` does not.

**Boundary:** `else if` is decided purely by the tokens that follow `else`. Because the parser sees no
newlines, an `else` whose *very next token* is `if` is **always** treated as an `else if` branch — you
cannot write an `else` block whose first statement is a plain `if`. (This is a rare, deliberate-looking
limitation of the token-driven grammar.)

---

## `while`

```jask
set i = 0
while i < 5
    printLine(i)
    set i = i + 1
endwhile
```

- The condition is re-evaluated before each iteration.
- `break` exits the loop; `continue` skips to the next iteration.

---

## `for...in`

```jask
for <variable> in <collection>
    <statements>
endfor
```

The loop variable is set (in the current scope) to each item of the collection.

### Iterating a list

```jask
for item in [1, 2, 3]
    printLine(item)
endfor
```

### Iterating a map

Each item is a **MapEntry** exposing `.key` and `.value`:

```jask
set m = { "x" = 10, "y" = 20 }
for entry in m
    printLine(entry.key + " -> " + entry.value)
endfor
```

### Iterating a string

Iterates by **Unicode code point** (runes); each item is a one-character string:

```jask
for ch in "café"
    printLine(ch)      ; c, a, f, é
endfor
```

### Boundaries

- A collection that is a list, map, or **string** is iterable; anything else errors:
  `'for...in' loop expects a list or a map, but got '{type}'`
  (strings are supported even though the message only mentions lists/maps).
- The loop variable: if it did **not** exist before the loop, it is removed afterward (no leak); if it
  shadowed an existing name, it keeps the last value after the loop.
- `break` / `continue` work inside `for...in` as in `while`.

---

## `break` and `continue`

| Keyword | Effect |
|---------|--------|
| `break` | exits the nearest enclosing `while` or `for...in` |
| `continue` | jumps to the next iteration of the nearest loop |

```jask
for i in listCreateRange(1, 10)
    if i % 2 == 0
        continue            ; skip evens
    endif
    if i > 7
        break               ; stop after 7
    endif
    printLine(i)
endfor
```

> `break`/`continue` must be inside a `while` or `for...in`. Using one at top level (or in a plain
> function) raises an uncaught control-flow exception, surfaced as `Unexpected error`.

---

## `return`

`return` ends a function and optionally yields a value.

```jask
function max(a: number, b: number)
    if a > b
        return a
    endif
    return b
endfunction
```

- **Only valid inside a function** — at top level (or inside `if`/`while` at top level) it is a parse
  error: `'return' is only allowed inside a function`.
- `return` with no value, or a function that reaches its end, returns **`nil`**.
- `return` exits the **entire function**, even from deeply nested `if`/`while`/`for` blocks.
- A function can have multiple `return` statements.

**Boundary:** a bare `return` (with no value) is only recognized when it is followed by one of
`endfunction`, `endif`, `endwhile`, `endfor`, `else`, `break`, `endtry`, or end-of-input. Because
`continue` and `catch` are not in that exclusion list, a bare `return` immediately followed by the
token `continue` or `catch` is a parse error (`Syntax Error continue: Expected an expression` /
`Syntax Error catch: …`). In practice this only matters if you put a bare `return` on the same logical
position directly before those tokens.

---

## Structuring notes

- Blocks are indented for readability, but indentation has **no** meaning to the parser.
- A block runs until its closing keyword. A missing closer is a syntax error, e.g.
  `Expected 'endif' at the end of the if statement`, `Expected 'endwhile' at the end of the while loop`,
  `Expected 'endfor' at the end of the for loop`, `Expected 'endfunction' at the end of the function`,
  `Expected 'endtry' at the end of the try block`.

---

## Next

- [Data Structures](05-data-structures.md) — lists, maps, structs.
- [Functions](06-functions.md) — the first-class workhorses.
- [Error Handling](08-error-handling.md) — `try`/`catch` for failing branches.
