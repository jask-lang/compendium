# Error Handling

[← Back to the Compendium index](README.md) · Prev: [Structs](07-structs.md) · Next: [Modules](09-modules.md)

jask distinguishes **recoverable** and **fatal** problems. This chapter covers the error model,
`try`/`catch`, and `assert`.
Source of truth: [`LangException.cs`](../interpreter/LangException.cs) and the `Statement.TryCatch`
case in [`Interpreter.cs`](../interpreter/Interpreter.cs).

---

## The error model

There are three ways a problem surfaces:

1. **Syntax / parse errors** — the code cannot be parsed. Thrown with the prefix `Syntax Error`:
   ```
   Error: [script.jask:3] Syntax Error =: Expected an expression
   ```
   These happen **before** any code runs, so a `try` block cannot catch them.

2. **Runtime interpreter errors** — represented by a `LangException`. These are the errors you **can**
   catch with `try`/`catch`. Examples:
   - `Unknown variable 'x'.`
   - `Division by zero`
   - `Operator '-' expects a number, but received 'abc'`
   - `Map does not contain key 'k'`
   - `Function 'f' has no overload that takes 1 argument(s)`
   - `Assertion failed`
   - `Missing permission 'read' for function 'readFile'`
   - … (every function's errors, catalogued in [Built-in Functions](11-built-in-functions.md)).

3. **Unexpected errors** — a .NET exception that is *not* a `LangException`. These are **not**
   catchable by `try`/`catch` and abort the script, printed as:
   ```
   Unexpected error: <.NET message>
   ```

### How an error is reported

When a `LangException` has a source location, it is rendered as `[<path>:<line>] <message>`; the line
is the source line where the error occurred. The interpreter prints `Error: <message>` in red to
stderr and stops (for a running script).

---

## `try` / `catch` / `endtry`

```jask
try
    set v = config["missing_key"]      ; raises an error
catch e
    printLine("caught: " + e.message)
    printLine("at line " + e.line)
endtry
```

- If the `try` body runs without a runtime error, the `catch` block is **skipped**.
- If the body raises a `LangException`, the `catch` block runs.
- The **error variable** after `catch` is **optional** (a bare `catch` is valid):

  ```jask
  try
      set x = 10 / 0
  catch
      set recovered = true
  endtry
  ```

- `try`/`catch` catches **runtime `LangException`s only**. Syntax errors and `.NET` ("Unexpected")
  errors are *not* caught.

### The error object

When you name the error variable (e.g. `catch e`), `e` is a **struct** of type `Error` with:

| Field | Type | Meaning |
|-------|------|---------|
| `e.message` | string | the error message (e.g. `Map does not contain key 'missing_key'`) |
| `e.line` | number | the source line where the error occurred |
| `e.file` | string / nil | the source file path (nil when not available) |

```jask
try
    set r = readFile("nope.txt")          ; (requires read permission)
catch e
    printLine(e.message)
endtry
```

> A subtle parse rule: the identifier after `catch` is treated as the error variable **only if it is
> not immediately followed by `(`** — otherwise it belongs to the first statement of the catch body.

---

## `assert`

`assert(condition)` fails (raises) when `condition` is not true:

```jask
assert(true)
assert(1 + 1 == 2)
assert(false)          ; Error: Assertion failed
```

- If the argument is not a boolean, it is a type error:
  `Function 'assert' expects a condition, but got '{type}'`.
- `assert` is the backbone of jask's testing style — see the [Testing conventions](13-advanced-topics.md#testing-conventions).

---

## Raising your own errors

There is no built-in `raise`/`throw` keyword. To signal a custom error, the common pattern is to raise
one of the built-in error conditions (e.g. read a bad map key), or to build a failing `assert`:

```jask
function divide(a: number, b: number)
    if b == 0
        assert(false)                 ; "Assertion failed"
    endif
    return a / b
endfunction
```

> For a cleaner, data-driven "error return" (instead of throwing), many jcore-style functions return a
> **`Result`** struct (`type`/`value`/`error`) — see
> [Security and Permissions → the Result struct](10-security-and-permissions.md#the-result-struct).

---

## Next

- [Modules](09-modules.md) — packaging code.
- [Security and Permissions](10-security-and-permissions.md) — the `Result` struct and the trust model.
- [Advanced Topics](13-advanced-topics.md) — how errors are attributed to files/lines.
