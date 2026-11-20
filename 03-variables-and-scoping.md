# Variables and Scoping

[← Back to the Compendium index](README.md) · Prev: [Language Basics](02-language-basics.md) · Next: [Control Flow](04-control-flow.md)

jask is dynamically typed: a variable holds **any** value, and a name can be (re)assigned a value of a
different type over its lifetime. Scope is lexical, implemented as a stack of environments
([`Interpreter.cs`](../interpreter/Interpreter.cs)).

---

## `set` — declaration and assignment

There is no separate "declare" and "assign" — `set` does both.

```jask
set x = 10          ; create (or overwrite) x in the current scope
set x = "ten"       ; re-assign a different type — allowed
set y = x + 1
```

- The right-hand side is evaluated first.
- `set x = v` writes to the **current (innermost) scope**. It never reaches into outer scopes.
- Names must start with a **lowercase** letter (or `_`).

> There is no "undeclared" error for reading: reading a name that exists only in an *outer* scope is
> fine (see [Lookup](#looking-up-variables)). Reading a name that exists **nowhere** is an error:
> `Unknown variable 'x'.`.

---

## The scope stack

- Every jask program has a **global** scope (the bottom of the stack).
- **Top-level code runs in the global scope.** So a top-level `set x = 1` *creates a global*.
- **Each function call** pushes a fresh local scope and pops it when the function returns.
- Struct definitions, and `interpret()`, also introduce temporary scopes (see
  [Structs](07-structs.md#defining-a-struct-type) and [Advanced Topics](13-advanced-topics.md#dynamic-code-interpret)).

```
[ function 3 scope ]   <- CurrentEnvironment (innermost)
[ function 2 scope ]
[ function 1 scope ]
[ global scope ]        <- _scopes[0], always present
```

### Looking up variables

A read searches the stack **from the innermost scope outward** and returns the first match
([`LookupVariable`](../interpreter/Interpreter/Interpreter.Expressions.cs)). If no scope has the name:

```
Unknown variable 'x'.
```

### Shadowing

Because `set` always writes to the innermost scope, a `set` inside a function **shadows** any global
of the same name — it does *not* modify the global:

```jask
set counter = 1

function bump()
    set counter = 99      ; NEW local counter; global is untouched
endfunction

bump()
printLine(counter)        ; 1
```

This is the key difference between **local** and **global** assignment.

### Shadowing built-ins

Built-in functions are **not** stored in any scope — a call is resolved by name against the function
tables, never through variable lookup. So a local can share a name with a built-in and shadow its
*reads* without affecting its *calls*:

```jask
set print = 5          ; a local variable named `print`
print                  ; 5   (variable lookup finds the local first)
print("hi")            ; still invokes the built-in `print`
```

Because functions are not first-class values (see [Functions](06-functions.md#function-visibility--semantics)),
there is no function value to capture or rebind — only name lookup and call resolution exist.

---

## Writing to globals: `set global`

From inside a function, use `set global` to write to the global scope:

```jask
set counter = 0

function increment()
    set global counter = counter + 1     ; writes the global
endfunction

increment()
increment()
printLine(counter)   ; 2
```

Rules / boundaries:

- The global must **already exist**: `set global fresh = 1` throws
  `Global variable 'fresh' is not defined`. (Create it at top level first.)
- A restricted global cannot be written: `Global variable 'x' is restricted and cannot be modified`.
- `set global` also works for field updates: `set global obj.field = value`
  (resolves `obj` in the global scope — see [Structs → field assignment](07-structs.md#field-assignment-is-a-value-update)).

> At top level there is only the global scope, so plain `set` and `set global` write to the same
> place. `set global` matters inside functions.

### Field assignment `set a.b.c = v`

You can write to a field (or a chain of fields) on a **struct** using the dotted form:

```jask
set p = Point(x = 1, y = 2)
set p.x = 99              ; rebinds `p` to a NEW instance with x = 99
set p.y = p.x + 1         ; chains resolve against the original binding
```

Resolution and boundaries (see [Structs → field assignment](07-structs.md#field-assignment-is-a-value-update)):

- The root variable is resolved innermost-first (or directly in the global scope for
  `set global a.b = v`); an unknown root is `Unknown variable 'a'.`.
- The value must be a **struct**: `Cannot set field 'b' on a non-struct value (got '{type}')` (maps,
  lists, numbers, strings, and `MapEntry` all fail here).
- Each field must exist: `Struct 'Point' has no field 'z'`.
- Because structs are immutable, this **rebinds the root variable to a new instance** — it is a value
  update, not an in-place mutation.
- Only the **root** name is case-checked; intermediate field names are not.

---

## `restrict` — making a variable immutable

`restrict` marks an existing variable so it can no longer be reassigned:

```jask
set pi = 3.14159
restrict pi

pi = 0          ; SyntaxError-free at parse, but at runtime:
                ; Variable 'pi' is restricted and cannot be modified
```

Boundaries:

- The variable must already be defined **in the current scope**: `Variable 'x' is not defined`.
- It must not already be restricted: `Variable 'x' is already restricted`.
- **Reading** a restricted variable works normally (the wrapper is transparent); only **writing** fails.
- Restricting a `nil` value is a no-op (nothing is wrapped).
- Useful for module-internal constants — e.g. `jcore/date.jask` does `set secondsPerDay = 86400` then
  `restrict secondsPerDay`.

---

## `nil` and uninitialized names

- A name is either bound (to some value, possibly `nil`) or unbound (reading it is an error).
- You can explicitly bind `nil`: `set x = nil`.
- Passing `nil` to a function, using `nil` as a map key, or as a map-literal value is an error
  (see the relevant chapters).

---

## Naming conventions

| Entity | Rule | Enforced? |
|--------|------|-----------|
| variable | start lowercase | yes (parse) |
| function / parameter | start lowercase | yes (parse) |
| module alias | start lowercase | yes (parse) |
| struct type | start uppercase | yes (parse) |
| struct field | any identifier | no |

Only the **first** character is validated, so `myVar`, `_x`, and `a1` are all valid.

---

## Next

- [Control Flow](04-control-flow.md) — `if`, `while`, `for...in`.
- [Data Structures](05-data-structures.md) — the values variables hold.
- [Structs](07-structs.md) — `set`-based field updates and immutability.
