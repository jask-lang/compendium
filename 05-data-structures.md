# Data Structures

[← Back to the Compendium index](README.md) · Prev: [Control Flow](04-control-flow.md) · Next: [Functions](06-functions.md)

jask has three value types that hold other values: **lists**, **maps**, and **struct instances**.
You also index into **strings** indirectly (via iteration / `charAt`). The full list/map function
catalogues are in [Built-in Functions](11-built-in-functions.md#list-functions) and
[… → Map functions](11-built-in-functions.md#map-functions).

---

## Lists

A **list** is an ordered, zero-indexed collection. Elements may be any type and may be nested.

### Creating

```jask
set a = [1, 2, 3]              ; list literal
set empty = []                 ; empty list
set mixed = [1, "two", true]   ; mixed types
set nested = [[1, 2], [3, 4]]
set b = list(1, 2, 3)         ; list(...) function (variadic)
```

- `list()` with no arguments → empty list.
- `list("abc")` with a **single string** argument → a list of its characters: `["a", "b", "c"]`
  (jask has no `char` type; characters are one-character strings). Use `listFrom` for the same.
  With any other shape of argument(s), `list(...)` simply collects the arguments in order.

**Literal boundaries:** a list literal is `[e1, e2, …]`. Elements are **expressions** (so you can nest
calls and structures). A **trailing comma is a syntax error** — `[1, 2, ]` fails with
`Expected an expression` (the parser expects another element). An empty list is `[]`.

### Access

```jask
set l = [10, 20, 30]
l[0]            ; 10          (bracket index)
listGet(l, 1)   ; 20
listSize(l)     ; 3
```

- Indices are **numbers**; negative or out-of-range indices are errors
  (`List index -1 out of range (list has 3 elements)`).

### Updating (immutable API)

All list "mutating" functions return a **new** list; the original is never changed:

```jask
set l = [1, 2, 3]
set l = listAdd(l, 4)          ; [1, 2, 3, 4]   (reassign to see the new list)
set l = listSet(l, 0, 99)      ; [99, 2, 3]
set l = listRemove(l, 0)       ; [2, 3]
set l = listReverse(l)         ; [3, 2, 1]
set l = listGetRange(l, 0, 1)  ; [3, 2]   (inclusive range)
set l = listExtend(l, [4, 5])  ; append a list
set r = listCreateRange(0, 5)  ; [0, 1, 2, 3, 4, 5]  (inclusive; descending if end < start)
```

> Because these return new values, you must **reassign** (e.g. `set l = listAdd(l, x)`). Forgetting
> to reassign is a common source of "the list didn't change" bugs.

---

## Maps

A **map** is a key → value collection. In literals, **keys must be string literals** and **values
cannot be `nil`**.

### Creating

```jask
set m = { "name" = "Alice", "age" = 30 }   ; map literal
set empty = {}                              ; empty map
set m2 = map(list("x", "y"), list(10, 20)) ; map(keys, values) function
```

Map-literal boundaries (all are errors):

| Condition | Error |
|-----------|-------|
| key not a string literal | `Map literal keys must be string literals` |
| value is `nil` | `Map literal values cannot be nil` |
| duplicate key | `Map literal has duplicate key 'k'` |

The map-literal syntax is exactly `{ "key" = value, "key2" = value2 }`: **string-literal keys**, the
`=` separator (not `:`), and a `}` closer. A **trailing comma is a syntax error** — `{ "a" = 1, }`
fails with `Expected a string key in map literal`. An empty map is `{}`.

### Access

```jask
set m = { "name" = "Alice", "age" = 30 }
m["name"]          ; "Alice"   (bracket key access)
mapGet(m, "age")   ; 30
mapHasKey(m, "age"); true
mapSize(m)         ; 2
mapGetKeys(m)      ; ["name", "age"]
mapGetValues(m)    ; ["Alice", 30]
```

- A **missing key is an error** (not a `nil` return): `Map does not contain key 'x'`.
- `nil` cannot be used as an index key: `Cannot use 'nil' as map index key`.

### Updating (immutable API, shallow copy)

```jask
set m = { "a" = 1, "b" = 2 }
set m = mapSet(m, "c", 3)      ; { "a" = 1, "b" = 2, "c" = 3 }
set m = mapRemove(m, "a")      ; { "b" = 2, "c" = 3 }   (removing a missing key is an error)
set merged = mapMerge(m1, m2)  ; on key collisions, the second map wins
```

- Like lists, map "mutators" return a **new** map (reassign to keep it).
- The copy is **shallow**: nested list/map values are shared by reference with the original.

---

## Struct instances

Structs are user-defined value types (see [Structs](07-structs.md) for definitions). Once created:

```jask
struct Point
    set x = 0
    set y = 0
endstruct

set p = Point(x = 1, y = 2)
p.x            ; 1          (member access)
type(p)        ; "Point"
```

Member access `p.field` returns the field's value; an unknown field is an error
(`Struct 'Point' has no member 'z'`).

---

## Indexing and member access

Two postfix access forms, chainable in any order ([`Parser.cs` `Call()`](../interpreter/Parser.cs)):

| Form | Works on |
|------|----------|
| `expr[key]` | **maps** (any key) and **lists** (numeric index) |
| `expr.field` | **struct instances** and **MapEntry** (`.key` / `.value`) |

```jask
m["a"]                ; map index
l[0]                  ; list index
p.x                    ; struct member
p.nested.x            ; chained member access
m["list"][0]          ; map → list → element
map["pair"].value     ; MapEntry member
```

- Indexing something that isn't a map or list: `Cannot index a '{type}' with brackets`.
- Member access on something that isn't a struct / MapEntry:
  `Attempted to access member 'f' on a non-struct value (got '{type}')`.

---

## Iteration

All three (plus strings) iterate with `for...in` — see [Control Flow → for...in](04-control-flow.md#forin):
lists yield elements, maps yield `MapEntry`s (`.key`/`.value`), strings yield one-character strings.

---

## Equality

`==` is structural for most values, **but maps are the exception** (verified against the runtime):

| Comparison | Result | Why |
|-----------|--------|-----|
| `[1,2] == [1,2]` | `true` | lists compare **deeply** (element-wise) |
| `{"a"=1} == {"a"=1}` | `false` | maps compare by **reference** (two distinct maps are not equal) |
| `m is m` | `true` | the same map object |
| `Point(1,2)` `==` equal struct | `true` | structs compare field-by-field (same type, same fields) |

Implication: to compare two maps by content, compare a sorted key list, or compare individual
values — `mapA["k"] == mapB["k"]`.

Nested values are compared recursively for lists and structs:

```jask
[1, [2, 3]] == [1, [2, 3]]   ; true
```

---

## Next

- [Functions](06-functions.md) — build and call behavior.
- [Structs](07-structs.md) — defining the third data type.
- [Built-in Functions](11-built-in-functions.md) — every list/map/struct function with exact errors.
