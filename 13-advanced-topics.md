# Advanced Topics

[← Back to the Compendium index](README.md) · Prev: [jcore Standard Library](12-jcore-standard-library.md)

This chapter covers dynamic code (`interpret`), how the interpreter is built, how errors are
attributed, and the language's known limitations and quirks.

---

## Dynamic code (`interpret`)

`interpret` runs a string of jask source at runtime and returns a value. It reuses the exact same
lexer → parser → executor pipeline as a normal script.

```jask
interpret("2 + 2")                       ; 4
interpret("1 + 1 * 2")                   ; 3
set r = interpret("
    set y = x + 1
    y * 100
")                                       ; 1100 (with x = 10 in scope)
```

### Semantics

- **Return value**: the value of the **last top-level expression** in the code, or `nil` if the code
  is empty or ends with a non-expression (e.g. `set`, `function`, `if`).
- **Scope**: runs in a **fresh local scope** on top of the current environment. It can **read** every
  existing variable, function, struct, and module. A plain `set` stays local and is discarded when
  `interpret` returns; `set global …` and `set existing.field = …` can still modify shared state.

  ```jask
  set x = 10
  interpret("set y = x + 1")   ; y is local to the call
  ; y is NOT defined here:
  interpret("y")               ; -> Unknown variable 'y'
  ```
- **Errors** (parse or runtime) **propagate** to the caller, so they are catchable:

  ```jask
  try
      set v = interpret("nope + 1")
  catch e
      printLine(e.message)
  endtry
  ```
- **Permissions**: inherits the host program's permission set — the interpreted code can do exactly
  what the host script can (and no more).
- **Nesting** works: `interpret("interpret(\"1 + 1\") + 2")` → `4`.

### Files

There is no `interpret(file = …)` form; load a file and pass its contents:

```jask
set src = readFile("snippet.jask")        ; (requires read permission)
if src.type == "OK"
    set result = interpret(src.value)
endif
```

> `interpret`'s own errors are attributed to the **host** file path (with the interpreted string's
> line number), since the snippet has no file of its own.

---

## Interpreter architecture

A jask program goes through a fixed pipeline:

1. **Lexer** ([`Lexer.cs`](../interpreter/Lexer.cs)) — turns source text into a token stream. Handles
   comments (`;`, `;;`), string escapes, numbers, identifiers/keywords. Tracks line numbers.
2. **Parser** ([`Parser.cs`](../interpreter/Parser.cs)) — builds an **AST** of `Statement`s and
   `Expression`s using a precedence-climbing expression grammar.
3. **Interpreter** ([`Interpreter.cs`](../interpreter/Interpreter.cs) + partials) — walks the AST and
   executes it.

Key interpreter state:

| Structure | Role |
|-----------|------|
| `_scopes` | a stack of environments; `_scopes[0]` is the global scope. `CurrentEnvironment` is the top. |
| `_functions` / `_functionOverloads` | the function table and per-name overload list. |
| `_structs` | struct type name → cached default field values. |
| `_modules` | module alias → the module's own `Interpreter`. |
| `_permissionManager` | the shared permission set (inherited by modules). |

- **Variables** resolve innermost-first across `_scopes`
  ([`LookupVariable`](../interpreter/Interpreter/Interpreter.Expressions.cs)).
- **Function calls** resolve against the function table, internal functions, and struct types, then
  match overloads by arity + type; the resolution is **cached on the AST node** and invalidated when
  new functions/structs are defined.
- **Modules** are separate `Interpreter` instances that share the permission manager; calls cross the
  module boundary as evaluated value literals.

---

## Error attribution

`LangException`s carry a source line (and, when available, a file path). At the top level the
interpreter renders them as `Error: [<path>:<line>] <message>` and stops.

- **Scripts**: the path is the script's file, the line is the source line.
- **Embedded jcore modules**: attributed as `[jcore/<path>:<line>]`.
- **`interpret`d code**: attributed to the host path with the snippet's line number (no separate file).
- **Built-in function** errors are attributed to the line of the function name in the call.

Two failure *classes* are reported differently: catchable `LangException`s print `Error: …`;
unexpected .NET exceptions print `Unexpected error: …` (and are not catchable). See
[Error Handling](08-error-handling.md).

---

## Testing conventions

jask tests are ordinary `.jask` files driven by the built-in `assert`:

```jask
printLine("=== section ===")
assert(1 + 1 == 2)
assert(someFunction(x) == expected)
printLine("passed")
```

- a failing `assert` raises `Assertion failed` and aborts the run (non-zero outcome);
- the jcore suite (`jcore/test/run_all_tests.jask`) `use`s each `test_*.jask` in turn;
- interpreter-level tests live in `interpreter/test/` (e.g. `test_interpret.jask`).

See [jcore Standard Library → testing](12-jcore-standard-library.md#testing-conventions).

---

## Limitations and known quirks

Documented faithfully from the current implementation — treat these as boundaries until changed:

- **Maps compare by reference** with `==`, while lists and structs compare deeply
  (`{ "a" = 1 } == { "a" = 1 }` is `false`). See [Data Structures → equality](05-data-structures.md#equality).
- **`+` is polymorphic**: number + number adds, anything else concatenates
  (`"5" + 5` → `"55"`). Convert with `toNumber`.
- **Only `nil` and `false` are falsy** — `0`, `""`, and `[]` are all truthy.
- **Functions are not hoisted** (define before you call) and are **not first-class**: no closures,
  anonymous functions, or passing a function as a value.
- **Structs are immutable**; `set obj.field = v` rebinds the variable to a new instance (value
  semantics), leaving other references to the old instance unchanged.
- **`set global x`** requires `x` to already exist globally; it cannot create new globals.
- **`break`/`continue`** outside a `while`/`for...in` raise an uncaught control-flow exception
  (`Unexpected error`).
- **`round`** is registered with one declared parameter but requires two arguments at runtime; named
  calls like `round(number = x)` are therefore inconsistent.
- **`http::post`** accepts a `body` argument but does not send it as request content; `get`/`post`
  have no explicit arity check (fewer arguments surface as an unexpected error).
- **`verify(value, "number")`** parses with the current culture, whereas `toNumber` uses the invariant
  culture.
- **`for...in`** accepts strings, though its error message only mentions lists and maps.
- **`getFields`**'s error message contains the source's spelling `"expets"`.
- **`list("abc")`** (a single string) yields characters `["a","b","c"]`, not `["abc"]`.
- **`print`/`printLine`** join arguments with no separator — use spaces explicitly or per-argument.
- **`date.jask`'s** `print`/`printLine` reference an undefined `stringFromDate` (a known library bug;
  they are untested and not relied upon).
- **Reserved struct names** `Result`, `MapEntry`, `HttpResponse` cannot be used for your own structs.
- **No statement-terminating semicolons**: newlines are not tokens, so statements are delimited by the
  grammar — keep one statement per line.
- **`else if` is token-adjacent only**: an `else` whose next token is `if` is *always* an `else if`
  branch, so an `else` block cannot begin with a plain `if`.
- **Bare `return` before `continue`/`catch`**: a valueless `return` immediately followed by the token
  `continue` or `catch` is a parse error (those tokens are missing from the "no-value" exclusion list).
- **`not` is not an arithmetic operand**: `not` binds looser than `==`/comparison/arithmetic, so it
  cannot be a direct operand of `+`/`-`/`<`/… — use parentheses: `a + (not b)`.
- **Number lexing**: a `.` is a decimal point only when followed by a digit, so `1.` lexes as `1` then
  `.`, and `1..2` lexes as `1 . . 2`.
- **`list` / `listFrom` share one implementation** with no type enforcement: `listFrom()` with zero
  args returns `[]`, and a single non-string arg is just wrapped (the "string" name is not enforced).
- **`mapSet` / `mapRemove` are shallow copies**: the new map shares nested list/map value references
  with the original.
- **Struct mutable defaults are shared**: a default that is a list/map is the *same object* behind
  every instance until a field is replaced (see [Structs](07-structs.md#defining-a-struct-type)).
- **Built-in shadowing**: `set print = 5` shadows *reading* `print` but `print("hi")` still calls the
  built-in (functions are not first-class values).
- **`restrict` on a `nil` value is a silent no-op** (the wrapper is only applied to non-nil values).
- **`verify(u, "string")` returns `Stringify(inner)`** (the stringified form), not the raw wrapped
  value.
- **`charCode`/`charFromCode` are UTF-16 code units**; `charCount`/`charAt` are Unicode code points —
  an emoji is 1 rune but 2 code units, so `charCode(emoji)` errors but `charCount(emoji)` counts it as 1.
- **List/map literals reject trailing commas**: `[1, ]` and `{ "a" = 1, }` are syntax errors.
- **A missing map key is an error, not `nil`**: `m["x"]` / `mapGet(m, "x")` raise
  `Map does not contain key 'x'` — check with `mapHasKey` first.

If you believe one of these is wrong, the source is authoritative — see the
[source map](README.md#how-to-use-this-compendium).

---

## Source map

| Area | File |
|------|------|
| Tokens & keywords | `interpreter/Token.cs`, `interpreter/Lexer.cs` |
| Grammar & precedence | `interpreter/Parser.cs` |
| Statements, scoping, control flow, modules | `interpreter/Interpreter.cs` |
| Expressions, operators, calls | `interpreter/Interpreter/Interpreter.Expressions.cs` |
| Built-in functions | `interpreter/Interpreter/Interpreter.InternalFunctions*.cs` |
| Trust/verify | `interpreter/Interpreter/Interpreter.TrustEngine.cs` |
| Helpers (Stringify, equality, type) | `interpreter/Interpreter/Interpreter.HelperFunctions.cs` |
| Struct / MapEntry data types | `interpreter/Interpreter/StructInstance.cs` |
| Permissions | `interpreter/Permissions.cs` |
| CLI entry / REPL | `interpreter/Program.cs`, `interpreter/Repl.cs` |
| Standard library | `jcore/math.jask`, `jcore/string.jask`, `jcore/date.jask` |
