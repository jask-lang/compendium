# Security and Permissions

[← Back to the Compendium index](README.md) · Prev: [Modules](09-modules.md) · Next: [Built-in Functions](11-built-in-functions.md)

jask is built around a **deny-by-default permission model** and a **trust boundary** for external
data. Source of truth: [`Permissions.cs`](../interpreter/Permissions.cs) and
[`Interpreter.TrustEngine.cs`](../interpreter/Interpreter/Interpreter.TrustEngine.cs).

---

## Permissions

A program starts with **no** permissions. Privileged built-ins check for a permission before acting
and fail (a catchable `LangException`) if it is missing.

| Permission name (in errors) | CLI flag | Gated function(s) |
|-----------------------------|----------|-------------------|
| `stdout` | `--allow-stdout` / `-ao` | `print`, `printLine` |
| `stdin` | `--allow-stdin` / `-ai` | `readInput` |
| `read` | `--allow-read` / `-ar` | `readFile`, `fileExists`, loading file-system modules |
| `write` | `--allow-write` / `-aw` | `writeFile` |
| `allow-network` | `--allow-network` / `-an` | `get`, `post` |
| `allow-trust-override` | `--allow-trust` / `-at` | `trust` only |

> **Naming quirk:** the four I/O permissions are named `stdout`/`stdin`/`read`/`write` in error
> messages, but the network and trust permissions keep their flag-style names
> (`allow-network`, `allow-trust-override`). `verify` and `untrust` need **no** permission.

### `--allow-all`

`--allow-all` grants **every** permission at once and short-circuits the other flags. It is not
mentioned in `--help` output.

### Missing a permission

Fails before the operation, e.g.:

```
Error: Missing permission 'read' for function 'readFile'
```

The exception is catchable by `try`/`catch`.

---

## Path-based read/write

`--allow-read` and `--allow-write` may be given **specific paths** (repeatable).

- **Bare flag (no paths)** → the base permission is granted for **all** paths:
  `dotnet run --allow-read ...` can read anything.
- **With paths** → access is restricted to those paths **and their sub-trees**:

  ```bash
  dotnet run --allow-read /home/user/data --input app.jask
  ```

  - `readFile("/home/user/data/notes.txt")` → **allowed** (under the path);
  - `readFile("/home/user/data")` → **allowed** (exact match);
  - `readFile("/home/user/data2/x")` → **not allowed** (prefix match is directory-aware — a trailing
    separator is enforced so `/data` never matches `/data2`).
  - Relative paths are resolved against the process working directory; paths are normalized
    (traversal-free) before comparison.

The check is done in two stages, so you may see either message:
`Missing permission 'read' for function 'readFile'` (no `read` at all) vs
`Missing permission 'read' on '<path>' for function 'readFile'` (read granted, but this path is not
covered).

---

## The `Result` struct

Many I/O and trust functions return a **`Result`** struct rather than throwing, so operational
failures (missing file, bad input, failed request) are data you can handle:

| Field | Type | Meaning |
|-------|------|---------|
| `type` | string | `"OK"` on success, `"ERR"` on failure |
| `value` | any | the payload (e.g. file contents, HTTP response), or `nil` |
| `error` | string | the error message, or `nil` |

```jask
set res = readFile("data.txt")
if res.type == "OK"
    printLine(res.value)
else
    printLine("failed: " + res.error)
endif
```

> `Result`, `MapEntry`, and `HttpResponse` are **reserved** struct names — you cannot define your own
> struct with those names. See [Error Handling](08-error-handling.md) for the separate `Error` struct
> (the `try`/`catch` error object), which is *not* a `Result`.

---

## Untrusted values and the trust boundary

Everything that crosses a trust boundary — **standard input**, **file contents**, **HTTP response
bodies** — is returned wrapped as an **untrusted** value:

- `type(untrusted)` → `"untrusted"`; it stringifies as `untrusted`.
- An untrusted value is **opaque**: you cannot use its contents directly. You must pass it through
  [`trust`](#trust) or [`verify`](#verify) to obtain a normal, typed value.

This is jask's answer to injection/unsanitized-input problems: external data arrives in a form that
forces you to either blindly trust it or validate it.

### The three trust functions

#### `trust` — the unsafe escape hatch

`trust(untrustedValue)` removes the wrapper and returns the raw value, **with no validation**. It
requires the `allow-trust-override` permission (`--allow-trust`).

```jask
; requires --allow-trust
set line = readInput("name?")
set name = trust(line.value)        ; no checking at all — your responsibility
```

- Errors: missing permission (`Missing permission 'allow-trust-override' for function 'trust'`),
  wrong arity, or argument not untrusted (`Function 'trust' expects an untrusted value but got '{type}'`).

#### `verify` — the safe path

`verify(untrustedValue, pattern)` validates the value against a pattern and returns a typed, trusted
value — **without** needing the trust permission. It returns a `Result`.

```jask
set line = readInput("age? ")
set v = verify(line.value, "number")
if v.type == "OK"
    printLine("age is " + v.value)
else
    printLine("invalid age: " + v.error)
endif
```

The value is converted to a string and **trimmed** before checking. The supported patterns are
**exact literal strings** (not regular expressions):

| Pattern | Accepts | Returned value |
|---------|---------|----------------|
| `"string"` | any non-nil value (no validation) | the stringified value |
| `"number"` | a parseable number | the parsed `number` |
| `"boolean"` | exactly `true`/`1` or `false`/`0` | `true` / `false` |
| `"json"` | any valid JSON document | JSON mapped to jask types (object→map, array→list, string→string, number→number, true/false→boolean, null→nil) |

- Failures are returned as `Result` (`type = "ERR"`), not thrown:
  - wrapped value is `nil` → `error: "Untrusted value for verify is nil"`
  - pattern unknown, or the value fails validation →
    `error: "Cannot verify value '{value}' for pattern '{pattern}'"`
- Thrown (argument) errors: wrong arity, first arg not untrusted, pattern not a string.

#### `untrust` — wrap a value

`untrust(value)` wraps **any** value (including `nil`) into a fresh untrusted value. Useful for testing
the trust workflow.

```jask
set u = untrust(42)
type(u)                 ; "untrusted"
verify(u, "number")      ; { type: "OK", value: 42, error: nil }
```

- Errors: wrong arity, or the value is already untrusted
  (`Function 'untrust' expects a value but got 'untrusted'`).

---

## A safe input workflow

```jask
set portText = readInput("port: ")
set portCheck = verify(portText.value, "number")

if portCheck.type != "OK"
    printLine("Port must be a number")
    exit(1)
endif

set port = portCheck.value
printLine("Using port " + port)
```

This never calls `trust`, so it does **not** require `--allow-trust`.

---

## Security notes

- **Default is deny**: a script with no flags cannot print, read, write, network, or trust.
- The permission set is **shared** across all modules in the program.
- `verify` is the recommended way to convert external data: it needs no privilege and enforces a
  shape. `trust` is a conscious, permission-gated "I accept this as-is" decision.
- Even a `Result` with `type == "OK"` carries its payload as an **untrusted** value (e.g.
  `readFile(...).value`, `get(...).value.body`) — verify or trust it before use.

---

## Next

- [Built-in Functions](11-built-in-functions.md) — the full function reference, with each function's permission requirements.
- [Advanced Topics](13-advanced-topics.md) — error attribution and limitations.
