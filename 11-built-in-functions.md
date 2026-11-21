# Built-in Functions

[← Back to the Compendium index](README.md) · Prev: [Security and Permissions](10-security-and-permissions.md) · Next: [jcore Standard Library](12-jcore-standard-library.md)

This is the complete catalogue of the **51** **interpreter** (built-in) functions. They are always
available — no `use` statement is needed — except the HTTP functions, which load via `use "jcore/http"`.

Every function that can fail reports an exact error message (a catchable `LangException`), usually
prefixed `Function '<name>' …`. `<type>` placeholders are the jask type name
(`number`, `string`, `boolean`, `list`, `map`, a struct name, `untrusted`, `nil`). Built-ins accept
**named arguments** (e.g. `readFile(file = "x")`); see [Functions → named arguments](06-functions.md#named-arguments).

Contents:
[Core](#core-functions) · [String and char](#string-and-char-functions) · [List](#list-functions) ·
[Map](#map-functions) · [I/O](#io-functions) · [HTTP](#http-functions) · [Module](#module-functions) ·
[Struct](#struct-functions) · [Trust](#trust-functions)

---

## Core functions

### `print(...)`
Prints the stringified arguments joined with **no separator**, no trailing newline. Returns `nil`.
*Requires* `stdout`. `print()` (no args) prints nothing.

### `printLine(...)`
Same as `print`, then emits a newline. Returns `nil`. *Requires* `stdout`.

### `clock()`
Returns the current Unix time in **seconds** as a `number` (millisecond precision). Returns a `number`.

### `type(value)`
Returns the type name as a `string`: `"number"`, `"string"`, `"boolean"`, `"list"`, `"map"`, a struct
type name, `"MapEntry"`, `"untrusted"`, or `"nil"`.

```jask
type(1)        ; "number"
type("a")      ; "string"
type([1])      ; "list"
type(Point())  ; "Point"
```

### `exit(code)`
Terminates the process with the given integer status code (truncated from a number). Never returns.
*Error:* `Function 'exit' expects an integer argument, but got '<type>'`.

### `assert(condition)`
Raises `Assertion failed` if `condition` is not true. *Errors:* `Function 'assert' expects a condition,
but got '<type>'`. See [Error Handling → assert](08-error-handling.md#assert).

### `sleepFor(seconds)`
Sleeps the current thread for `seconds` (fractional ms truncated). Returns `nil`.
*Error:* `Function 'sleepFor' expects a number argument, but got '<type>'`.

### `interpret(code)`
Runs `code` (a `string`) as jask source in the current environment and returns the value of the last
top-level expression (or `nil`). *Requires* `interpret` (`--allow-interpret` / `-ae`). Function and
struct definitions made inside `code` do **not** leak back to the caller. See
[Advanced Topics → interpret](13-advanced-topics.md#dynamic-code-interpret).
*Errors:* `Missing permission 'interpret' for function 'interpret'`;
`Function 'interpret' expects a string argument, but got '<type>'`.

### `toNumber(value)`
Converts a `number` (unchanged) or a numeric `string` to a `number`.
*Errors:* `Function 'toNumber' could not convert string '<s>' to a number`;
`Function 'toNumber' expects a number or string argument, but got '<type>'`.

### `toString(value)`
Returns the string form of a value (via the same `Stringify` rules used by `print`).

### `round(value, digits)`
Rounds `value` to `digits` decimal places (banker's rounding — halves round to even — and `digits`
truncated to an integer). Returns a `number`.

```jask
round(2.5)        ; 2   (banker's rounding: .5 → even)
round(3.5)        ; 4
round(3.14159, 2) ; 3.14
```

*Errors:* `Function 'round' expects 2 argument(s), but got <n>`; `Function 'round' expects a number
argument, but got '<type>'` (and `… for digits …` if the second argument isn't a number).

> **Quirk:** `round` is *registered* with a single declared parameter but its implementation requires
> two arguments, so named calls like `round(number = x)` are inconsistent with the signature.

### `floor(value)` / `ceil(value)`
Round down / up to the nearest integer. Returns a `number`.
*Error:* `Function 'floor' expects a number argument, but got '<type>'` (and `ceil`).

---

## String and char functions

Two families exist: **char-level** (single UTF-16 code unit) and **code-point-level** (Unicode runes).

| Function | Family |
|----------|--------|
| `charCode`, `charToUpper`, `charToLower` | single code unit (must be length 1) |
| `charCount`, `charAt` | Unicode code points (runes) |

### `charCode(ch)`
The code unit of a **single-character** string, as a `number`.
*Errors:* `Function 'charCode' expects a string argument, but got '<type>'`;
`Function 'charCode' expects a single-character string, but got '<s>'` (a surrogate pair / emoji is rejected).

### `charFromCode(code)`
The single character (as a `string`) for a Unicode code point. The number is truncated to an int;
valid points are `0..0x10FFFF` excluding surrogates.
*Errors:* `Function 'charFromCode' expects a number argument, but got '<type>'`;
`Function 'charFromCode' expects a valid Unicode code point, but got <code>`.

### `charToUpper(ch)` / `charToLower(ch)`
The upper / lower case of a single-character string. *Errors:* as `charCode`.

### `charCount(s)`
The number of **code points** (runes) in `s`. An emoji counts as 1. Returns a `number`.
*Error:* `Function 'charCount' expects a string argument, but got '<type>'`.

### `charAt(s, index)`
The rune at `index` (0-based, code-point based) as a one-character `string`.
*Errors:* `Function 'charAt' expects second argument to be a number, but got '<type>'`;
`Function 'charAt' index <i> is out of bounds for string of length <n>`.

> For higher-level string operations (length, substring, split, join, replace, trim, …) use the
> jcore **`string`** module — [jcore Standard Library](12-jcore-standard-library.md#module-string).

---

## List functions

### `list(...)` / `listFrom(...)`
- `list(a, b, …)` → a list of the arguments; `list()` → `[]`.
- `list("abc")` (a single string argument) → the characters as a list: `["a", "b", "c"]`.
- `listFrom` behaves identically (no extra type enforcement).

### `listSize(l)`
The element count. *Error:* `Function 'listSize' expects a list, but got '<type>'`.

### `listGet(l, i)` / bracket access `l[i]`
The element at index `i`. Out of range / bad index:
`Function 'listGet' index <i> is out of bounds for list of size <n>`;
`List index must be a number, but got '<type>'`.

### `listAdd(l, x)` / `listSet(l, i, x)` / `listRemove(l, i)` / `listReverse(l)`
Each returns a **new** list (append / replace / remove / reverse); the input is unchanged.
*Errors:* `Function 'listAdd' expects first argument to be a list, but got '<type>'` (same shape for
the others); out-of-range variants for `listSet` / `listRemove`.

### `listGetRange(l, start, end)`
A new sub-list from `start` to `end`, **inclusive**.
*Error:* `Function 'listGetRange' indices [<start>, <end>] are out of bounds for list of size <n>`
(if `start < 0`, `end >= size`, or `start > end`).

### `listExtend(l, other)`
Both lists → a new list with `other` appended; both strings → their concatenation.
*Error:* `Function 'listExtend' expects both arguments to be lists or both to be strings, but got '<t1>' and '<t2>'`.

### `listCreateRange(start, end)`
A list stepping by 1 from `start` to `end`, **inclusive**; descending if `end < start`. Fractional
bounds are honored (`listCreateRange(1.5, 3)` → `[1.5, 2.5, 3]`).
*Errors:* `Function 'listCreateRange' expects first argument to be a number, but got '<type>'`
(and second).

> All mutating functions return new lists — reassign to keep them
> (`set l = listAdd(l, x)`). See [Data Structures → lists](05-data-structures.md#lists).

---

## Map functions

### `map(keys, values)`
Builds a map from two lists (string keys, non-nil values, equal lengths). *Errors:*
`Function 'map' expects a list, but got '<type>'`; `Function 'map' does not allow 'nil' as key`;
`Function 'map' expects a unique list of keys. Value '<k>' is not unique in key list`;
`Function 'map' expects a list of strings for keys, but got '<type>'`;
`Function 'map' expects the same amount of keys and values`;
`Function 'map' does not allow 'nil' as value for key`.

### `mapGet(m, k)` / bracket access `m["k"]`
The value for `k`. A missing key is an **error**, not `nil`:
`Function 'mapGet' cannot find key '<k>'` / `Map does not contain key '<k>'`.

### `mapSet(m, k, v)`
A new map with `k` set/updated (shallow copy). `nil` value is an error:
`Function 'mapSet' cannot use 'nil' as value`.

### `mapGetKeys(m)` / `mapGetValues(m)`
A list of the map's keys / values. *Error:* `Function 'mapGetKeys' expects a map, but got '<type>'`.

### `mapHasKey(m, k)` / `mapSize(m)`
Whether `k` is present (bool) / the key count (number). *Error:* `Function 'mapSize' expects a map, but got '<type>'`.

### `mapRemove(m, k)`
A new map without `k`. Removing a missing key is an error: `Function 'mapRemove' cannot find key '<k>'`.

### `mapMerge(m1, m2)`
A new merged map; on key collisions **`m2` wins**. *Error:* `Function 'mapMerge' expects a map, but got '<type>'`.

> Maps compare by **reference** with `==` — see [Data Structures → equality](05-data-structures.md#equality).

---

## I/O functions

`readInput`, `readFile`, and `writeFile` return a **`Result`** struct; `fileExists` returns a plain
boolean. (HTTP `get`/`post` and `verify` also return a `Result`.)

**The `Result` struct** (reserved name; full model in
[Security and Permissions → the Result struct](10-security-and-permissions.md#the-result-struct)):

| Field | Type | Meaning |
|-------|------|---------|
| `type` | string | `"OK"` on success, `"ERR"` on failure |
| `value` | any | the payload, or `nil` on failure |
| `error` | string / nil | the error message, or `nil` on success |

```jask
set res = readFile("data.txt")
if res.type == "OK"
    ; res.value is an *untrusted* string — verify or trust it before use
else
    printLine("failed: " + res.error)
endif
```

### `readInput(prompt)` — *requires* `stdin`
Reads one line from stdin (optionally printing `prompt` first). On success `value` is an
**untrusted** string. *Errors:* `Missing permission 'stdin' for function 'readInput'`;
`Function 'readInput' expects 0 or 1 argument, but got <n>`. *Result failure:* `error: "Reading input failed"`.

### `readFile(file)` — *requires* `read`
Reads the whole file. `value` is an **untrusted** string. *Errors:*
`Missing permission 'read' for function 'readFile'`; `Function 'readFile' expects a string argument, but got '<type>'`;
`Missing permission 'read' on '<path>' for function 'readFile'`. *Result failures:*
`error: "File at path '<path>' cannot be found"`; `error: "Reading file at '<path>' failed"`.

### `writeFile(file, content)` — *requires* `write`
Writes (creates/overwrites) a file. `value` is `nil` on success. *Errors:*
`Missing permission 'write' for function 'writeFile'`;
`Function 'writeFile' expects a string argument for path, but got '<type>'`;
`Function 'writeFile' expects a string argument for content, but got '<type>'`;
`Missing permission 'write' on '<path>' for function 'writeFile'`. *Result failure:*
`error: "Writing file at '<path>' failed"`.

### `fileExists(file)` — *requires* `read`
Whether the file exists (bool). *Errors:* as `readFile` (same permission and arity messages).

---

## HTTP functions

Available only after `use "jcore/http" as http` and *requiring* `allow-network`.
`get` / `post` return a `Result` whose `value` (on success) is an **`HttpResponse`** struct.

### `http::get(url, headers)`
Sends a GET request. `headers` is a map of string → string. *Errors:*
`Missing permission 'allow-network' for function 'get'`; `Function 'get' expects a string but got '<type>'`;
`Function 'get' expects a map, but got '<type>'`;
`Function 'get' expects a map with string values, but got value of type '<type>'`.
*Result failure:* `error: "Function 'get' failed to fetch '<uri>': <message>"`.

### `http::post(url, headers, body)`
Sends a POST request (same shape and errors as `get`, with `post` in messages).

> **Quirk (faithful to the current implementation):** the `body` argument is accepted but not
> actually sent as request content, and there is no explicit arity check (passing fewer than two
> arguments surfaces as an unexpected error).

### The `HttpResponse` struct

| Field | Type |
|-------|------|
| `statusCode` | number |
| `body` | untrusted string |
| `headers` | map (string → string; multi-value headers joined with `", "`) |

```jask
use "jcore/http" as http
set res = http::get("https://example.com", {})
if res.type == "OK"
    printLine(res.value.statusCode)
endif
```

---

## Module functions

### `unfoldModule(alias)`
Returns a `string` documenting a previously loaded module: its loaded sub-modules, its struct
definitions, and its function signatures (including parameter defaults).
*Errors:* `Function 'unfoldModule' expects a module alias or a string matching a module alias, but got '<type>'`.

---

## Struct functions

### `getFields(struct)`
A list of the instance's field names. *Errors:* `Function 'getFields' expets a struct instance, but got '<s>'`
(note the source's "expets" spelling).

### `hasField(struct, name)`
Whether the instance has `name` (bool). *Errors:*
`Function 'hasField' expects a struct instance, but got '<s>'`; `Function 'hasField' expects a string, but got '<s>'`.

---

## Trust functions

See [Security and Permissions](10-security-and-permissions.md#untrusted-values-and-the-trust-boundary) for
the full trust model.

### `trust(untrustedValue)` — *requires* `allow-trust-override`
Unwraps an untrusted value into a normal value with **no validation**.
*Errors:* `Missing permission 'allow-trust-override' for function 'trust'`;
`Function 'trust' expects an untrusted value but got '<type>'`.

### `verify(untrustedValue, pattern)`
Validates an untrusted value against `"string"` / `"number"` / `"boolean"` / `"json"` and returns a
`Result` with a typed trusted value. No permission required.
*Errors:* `Function 'verify' expects an untrusted value but got '<type>'`;
`Function 'verify' expects a string pattern but got '<type>'`. *Result failures:*
`error: "Untrusted value for verify is nil"`; `error: "Cannot verify value '<v>' for pattern '<p>'"`.

### `untrust(value)`
Wraps any value (including `nil`) into a new untrusted value.
*Error:* `Function 'untrust' expects a value but got 'untrusted'`.

---

## Next

- [jcore Standard Library](12-jcore-standard-library.md) — the `math`, `string`, `date` modules.
- [Advanced Topics](13-advanced-topics.md) — `interpret`, architecture, and known quirks.
