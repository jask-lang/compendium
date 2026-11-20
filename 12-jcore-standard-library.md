# jcore Standard Library

[← Back to the Compendium index](README.md) · Prev: [Built-in Functions](11-built-in-functions.md) · Next: [Advanced Topics](13-advanced-topics.md)

**jcore** is the standard library, written in jask and embedded in the interpreter. It has three
modules — `math`, `string`, `date` — each imported with `use` and called through an alias with `::`.

```jask
use "jcore/math" as math        ; or "jcore/math.jask" — the extension is optional
printLine(math::max(1, 2))       ; 2
```

Because they are embedded `jcore/…` modules, importing them needs **no** `read` permission.

---

## Module `math`

`use "jcore/math" as math`

| Function | Returns | Behavior |
|----------|---------|----------|
| `max(n1, n2)` | number | the larger; ties return `n1` |
| `min(n1, n2)` | number | the smaller; ties return `n1` |
| `absolute(n)` | number | `n < 0 ? -n : n` |
| `power(base, exponent)` | number | `base^exponent` (binary exponentiation) |
| `root(x, n)` | number / nil | the n-th root of `x` (Newton's method) |
| `squareroot(x)` | number / nil | `root(x, 2)` |
| `fibonacci(n)` | number | the n-th Fibonacci number (iterative) |
| `isPrime(n)` | boolean | wheel-factorization primality test |

### `power` edge cases
- `power(0, 0)` → `1` (treated as 1).
- `power(0, -2)` → `nil` (zero to a negative power).
- A **negative exponent** returns a float: `power(-2, -3)` → `-0.125`.

```jask
math::power(base = 2, exponent = 4)      ; 16
math::power(base = 0, exponent = -2)     ; nil
```

### `root` edge cases (approximate results)
- `root(x, 0)` → `nil` (0th root undefined).
- `root(x, n)` for an **even** `n` with `x < 0` → `nil` (no real root).
- `x == 0` with `n > 0` → `0`.
- Negative `x` with odd `n` → the negative root.
- Results are **approximate** (IEEE-754); compare with a tolerance, not `==`:

  ```jask
  function isClose(a: number, b: number) return math::absolute(a - b) < 0.00001 endfunction
  isClose(math::root(8, 3), 2.0)         ; true
  ```

### `fibonacci` and `isPrime`
- `fibonacci(n)`: `n <= 1` → `n` (so negative input returns `n`); otherwise iterative.
  `fibonacci(30)` → `832040`.
- `isPrime(n)`: `false` for `n <= 1` and even multiples; `true` for `2`, `3`; otherwise a 6k±1 wheel
  up to `√n`. `isPrime(19)` → `true`, `isPrime(-5)` → `false`.

---

## Module `string`

`use "jcore/string" as str` — the **Unicode-aware** string toolkit. Indexing/iteration is by
**code point** (so `"café"` has length 4, not 5). Built on the built-in char/charCount functions.

| Function | Returns | Behavior |
|----------|---------|----------|
| `length(s)` | number | number of code points |
| `indexOf(s, sub)` | number | first code-point index of `sub`, or `-1` (empty `sub` → `0`) |
| `substring(s, i, len)` | string | substring from `i` of length `len` (clamped; out of range → `""`) |
| `toUpper(s)` / `toLower(s)` | string | case-map every code point |
| `trim(s)` / `trimLeft(s)` / `trimRight(s)` | string | strip whitespace (space/tab/LF/CR) |
| `split(s, delim)` | list | split on `delim`; empty `delim` → `[s]`; trailing/consecutive delimiters yield empty elements |
| `join(list, delim)` | string | join with `delim` (elements stringified) |
| `contains(s, sub)` | boolean | `indexOf != -1` (so `contains(s, "")` is true) |
| `startsWith(s, p)` / `endsWith(s, p)` | boolean | prefix/suffix check (empty pattern → true) |
| `replace(s, old, new)` | string | replace **all** non-overlapping occurrences |
| `replaceFirst(s, old, new)` | string | replace only the first occurrence |
| `count(s, sub)` | number | non-overlapping occurrences (empty `sub` → `0`) |
| `padLeft(s, w, c)` / `padRight(s, w, c)` | string | pad to width `w` with `c` (empty `c` → space) |
| `padCenter(s, w, c)` | string | center; odd padding goes on the **right** |
| `repeat(s, n)` | string | repeat `n` times (`n <= 0` → `""`) |
| `reverse(s)` | string | reverse code-point order |
| `isEmpty(s)` | boolean | `length(s) == 0` |
| `isAlpha(s)` | boolean | non-empty, all letters |
| `isNumeric(s)` | boolean | non-empty, all ASCII digits `0-9` (no sign/decimal) |
| `isAlphanumeric(s)` | boolean | non-empty, all letters or ASCII digits |
| `isWhitespace(s)` | boolean | non-empty, all space/tab/LF/CR |
| `capitalize(s)` | string | uppercase only the first character |
| `titleCase(s)` | string | capitalize first letter of each word, lowercase the rest |
| `stringFormat(template, args)` | string | replace `{0}`, `{1}`, … with the argument string forms |

Examples:

```jask
str::length("café")                       ; 4
str::indexOf("café", "fé")                 ; 2
str::split("a,b,c", ",")                   ; ["a", "b", "c"]
str::join(list("a", "b"), " - ")           ; "a - b"
str::replace("aaa", "a", "bb")             ; "bbbbbb"
str::stringFormat("Hi {0}, {1}!", list("A", 30))  ; "Hi A, 30!"
str::titleCase("café naïve")               ; "Café Naïve"
```

**`stringFormat` boundary:** replacements run sequentially over the whole string, so an inserted value
that itself contains a later placeholder gets replaced too —
`stringFormat("{0} {1}", list("{1}", "x"))` → `"x x"`. Unused placeholders remain as-is and extra
`args` are ignored.

---

## Module `date`

`use "jcore/date" as date` — calendar-date arithmetic. A **`Date`** is a struct of
`day`, `month`, `year` (a calendar date, **not** a count of seconds). Conversions to/from the
epoch are **day-granular** (`86400` s/day; no time-of-day).

```jask
use "jcore/date" as date
set d = date::Date(year = 2026, month = 1, day = 31)
printLine(d.year)        ; 2026
```

Defaults: an omitted field defaults to `1` (so the epoch default is `1970-01-01`).

| Function | Returns | Behavior |
|----------|---------|----------|
| `checkIfYearIsLeapYear(year)` | boolean | Gregorian leap-year rule |
| `isValid(date)` | boolean | range checks + a round-trip sanity check |
| `getDateFromElapsedSeconds(s)` | Date | seconds since 1970-01-01 → a Date (sub-day seconds dropped) |
| `getElapsedSecondsFromDate(d)` | number | a Date → whole-day seconds since 1970-01-01 |
| `isDateAfter(d1, d2)` / `isDateBefore(d1, d2)` | boolean | strict comparison (equal → false) |
| `addDays(d, n)` / `subtractDays(d, n)` | Date / nil | `n < 1` → `nil` |
| `getDifferenceBetweenDates(d1, d2)` | number | whole-day multiple of `86400` (positive if `d1` is later) |
| `getDayOfWeek(d)` | number | `1`=Monday … `7`=Sunday |
| `getDayOfYear(d)` | number | 1..365/366 (leap-aware) |
| `getWeekOfYear(d)` | number | ISO 8601 week (1..53) |
| `getDateFromString(s, format)` | Date / nil | parse using `YYYY`/`MM`/`DD` markers in `format` |
| `getStringFromDate(d)` | string | format as `YYYY-MM-DD` (year not zero-padded) |
| `print(d)` | — | **buggy** — body calls an undefined `stringFromDate`; see note below |
| `printLine(d)` | — | **buggy** — relies on the broken `print` above; see note below |

Examples:

```jask
date::checkIfYearIsLeapYear(2000)                 ; true
date::getDateFromElapsedSeconds(0).year           ; 1970
date::getElapsedSecondsFromDate(date::Date(year = 2000, month = 1, day = 4))   ; 946944000
date::getDayOfWeek(date::Date(year = 1960, month = 7, day = 11))               ; 1
set s = date::getStringFromDate(d)                ; "2026-01-31"
date::getDateFromString(s, format = "YYYY-MM-DD") ; round-trips to d
date::addDays(d, -1)                              ; nil
```

> `getDateFromString` aligns the `YYYY`/`MM`/`DD` markers in `format` **positionally** with the fields
> in the input string. **Boundary:** if the input string is shorter than the marker's position+length,
> the slice is `""` and `toNumber("")` **throws** `Function 'toNumber' could not convert string '' to
> a number` (a runtime error, not a `nil` result).

> **Known library bug:** `date::print` and `date::printLine` are exported but reference a function
> named `stringFromDate` that does not exist (the real name is `getStringFromDate`). Calling them
> raises `Unknown function 'stringFromDate'`. Use `date::getStringFromDate(d)` + the built-in
> `printLine` instead. These two are untested and should not be relied upon.

---

## Testing conventions

The jcore test suite (`jcore/test/`) shows jask's idiomatic test style (used by the built-in `assert`):

- one `assert(<boolean>)` per check; compound checks use `and`;
- each section: `printLine("=== <name> ===")` … `printLine("passed")`;
- float comparisons use a local `isClose` helper (since `root`/`squareroot` are approximate);
- `nil` results are checked with `== nil`; keyword-argument calls are common
  (`math::power(base = 2, exponent = 4)`);
- the whole suite runs with `run_all_tests.jask` (see [Getting Started](01-getting-started.md#running-jask)).

---

## Next

- [Advanced Topics](13-advanced-topics.md) — `interpret`, architecture, and limitations.
