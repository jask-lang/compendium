# Structs

[← Back to the Compendium index](README.md) · Prev: [Functions](06-functions.md) · Next: [Error Handling](08-error-handling.md)

Structs are jask's user-defined **value types**: a named bundle of fields with default values.
Source of truth: the `Statement.Struct` case in [`Interpreter.cs`](../interpreter/Interpreter.cs) and
[`StructInstance.cs`](../interpreter/Interpreter/StructInstance.cs).

---

## Defining a struct type

```jask
struct Point
    set x = 0
    set y = 0
endstruct
```

- The type name must start with an **uppercase** letter.
- The body is a list of `set <field> = <default>` statements. Fields may be any type; defaults may be
  nested values.
- A struct type is registered in the current module; redefining a name is an error
  (`Struct 'Point' is already defined`).
- Three names are **reserved** and cannot be used as struct types: `Result`, `MapEntry`,
  `HttpResponse` (they are the runtime's own struct types).
- Defaults are evaluated **once, at definition time**, in a temporary scope. A field can reference an
  earlier field's default.

> **Mutable-default footgun:** because defaults are evaluated once and every instance starts from the
> same default *value objects*, a default that is a **list** or **map** is shared by reference across
> all instances until a field is replaced. This bites if you ever mutate the default value in place —
> prefer immutable (number/string/boolean) defaults, or replace the field with a fresh collection:
>
> ```jask
> struct Bag
>     set items = []        ; the SAME empty list object backs every instance
> endstruct
> ```

> A struct **definition** creates the type; it does **not** create an instance. `struct Point` on its
> own does not bind a variable named `Point`.

---

## Creating instances

Instances are created by calling the type name with **named** fields:

```jask
set p1 = Point()                    ; all defaults: { x: 0, y: 0 }
set p2 = Point(x = 3)               ; { x: 3, y: 0 }  (y keeps its default)
set p3 = Point(x = 1, y = 2)
```

- **Named fields only.** Positional arguments are an error:
  `Struct 'Point' instantiation with positional arguments is not supported. Use named fields: Point(field = value, ...)`.
- Any omitted field takes its default.
- An unknown field is an error: `Struct 'Point' has no field 'z'`.
- `type(p3)` → `"Point"`.

---

## Reading fields

```jask
set p = Point(x = 1, y = 2)
p.x        ; 1
p.y        ; 2
p.z        ; Struct 'Point' has no member 'z'
```

Chained member access works for nested structs: `outer.inner.field`.

---

## Field assignment is a value update

`set obj.field = value` does **not** mutate in place. Structs are **immutable**: the statement
produces a **new** struct instance with the updated field and **rebinds the variable** to it.

```jask
struct Counter
    set value = 0
endstruct

set a = Counter()
set b = a
set b.value = 5        ; b now points to a NEW instance; a is untouched

printLine(a.value)     ; 0
printLine(b.value)     ; 5
printLine(a is b)      ; false   (now distinct instances)
printLine(a == b)      ; false   (fields differ)
```

Consequences to remember:

- Other variables that referred to the old instance **still point to it**. Updating `b` above did not
  change `a`.
- Chained updates work and rebind from the top: `set obj.inner.field = 9` rebuilds `inner`, then
  `obj`. Each intermediate level must itself be a struct
  (`Cannot set field 'f' on a non-struct value (got '{type}')`).
- `set global obj.field = value` resolves `obj` in the **global** scope instead of the current one.
- Because this rebinds a variable, it's only meaningful for variables you can write (see
  [Variables and Scoping](03-variables-and-scoping.md#shadowing)).

> This is the single most surprising struct behavior. If you want two names to track the same changing
> value, reassign both: `set a = updated; set b = updated`.

---

## Introspection

| Function | Returns |
|----------|---------|
| `getFields(obj)` | list of the instance's field names |
| `hasField(obj, "name")` | whether the instance has that field |

```jask
set p = Point(x = 1, y = 2)
getFields(p)          ; ["x", "y"]
hasField(p, "x")      ; true
hasField(p, "z")      ; false
```

---

## Equality

- `==` compares structs **by value**: same type name, same set of fields, field-by-field equality.
- `is` compares by **instance** (reference) — two `Point(x = 1)` created separately are `==` but not
  `is`.

```jask
Point(x = 1, y = 2) == Point(x = 1, y = 2)   ; true
Point(x = 1) == Point(x = 2)                  ; false
```

---

## Next

- [Error Handling](08-error-handling.md) — structs that report failures (`Result`), and catching the errors structs raise.
- [Modules](09-modules.md) — `export struct` and using struct types across modules (`mod::StructName`).
