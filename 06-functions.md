# Functions

[← Back to the Compendium index](README.md) · Prev: [Data Structures](05-data-structures.md) · Next: [Structs](07-structs.md)

Functions are the main unit of behavior in jask. They are defined with `function`, called by name, and
support typed parameters, default values, overloading, and named arguments.
Source of truth: [`Parser.cs`](../interpreter/Parser.cs) and
[`Interpreter.Expressions.cs`](../interpreter/Interpreter/Interpreter.Expressions.cs).

---

## Definition

```jask
function add(a: number, b: number)
    return a + b
endfunction

printLine(add(2, 3))   ; 5
```

- Every parameter **must** carry a type annotation (`name: type`).
- A function name must start with a **lowercase** letter.
- The body runs in a **fresh local scope**; local variables do not leak to the caller.
- A **single-line** form is available:

  ```jask
  function double(x: number) return x * 2 endfunction
  ```

- `export function` marks a function for use by other modules (see [Modules](09-modules.md#exports)).

### Parameter types

| Type token | Accepts |
|------------|---------|
| `number` | any number |
| `string` | any string |
| `boolean` | `true` / `false` |
| `list` | any list |
| `map` | any map |
| `any` | **anything** (no type checking) |
| `<StructName>` | a struct instance of that type (e.g. `Date`) |

### Default values

A parameter may declare a default, which is used when the argument is omitted:

```jask
function greet(name: string, prefix: string = "Hello")
    return prefix + ", " + name
endfunction

greet("Alice")              ; "Hello, Alice"
greet("Alice", "Hey")       ; "Hey, Alice"
greet(name = "Alice")       ; "Hello, Alice"   (named)
```

- Defaults are evaluated at call time for each omitted parameter.
- A missing **required** parameter (no default, not supplied) is an error:
  `Missing required parameter 'b' when calling 'add'`.

---

## Overloading

The same function name may have **multiple** definitions with different parameter signatures.

```jask
function greet(name: string)
    return "hi " + name
endfunction

function greet(x: number, y: number)
    return "sum " + (x + y)
endfunction

greet("bob")    ; "hi bob"   (matched by: 1 arg, string)
greet(1, 2)     ; "sum 3"    (matched by: 2 args, number number)
```

- Overload identity is **name + parameter types**. Redefining the same name **with the same
  parameter types** is an error:
  `Function 'add' with the same parameter types is already defined`.
- **Positional** calls are matched by argument count and types. The best overload is the one with the
  most **required** (non-default) parameters; ties are broken by the fewest `any` parameters.
- `any` parameters match any value.

If no overload matches:

- no overload has a compatible argument count →
  `Function 'name' has no overload that takes {n} argument(s)`
- an overload has the right count but the types don't fit →
  `Function 'name' has no overload matching types ({type1}, {type2}, …)`
- the name is unknown entirely → `Unknown function 'name'`

---

## How a call is resolved

For a given call name the interpreter consults **three tables at once** — user functions
(overloads), built-in (internal) functions, and struct types — then picks a target. The exact order
matters and differs between **positional** and **named** calls:

### Positional call `name(args)`

1. If only a built-in is registered under the name → call the built-in directly.
2. If a **struct type** has the name → instantiate it (0 args → all defaults; any positional args →
   error `Struct 'Name' instantiation with positional arguments is not supported. Use named fields: …`).
   A struct **shadows** a same-named user function for positional calls.
3. Otherwise, evaluate arguments and match a user overload by count + type (see above). If no user
   overload matches but a built-in of the same name exists, the built-in is used as a fallback.

### Named call `name(p = v)`

1. If a **built-in** is registered under the name → call it, **regardless of user overloads** (built-ins
   win for named calls even when a same-name user function exists).
2. If a struct type has the name → instantiate with named fields.
3. Otherwise match a user overload by the supplied parameter **names** (each must be a declared
   parameter; omitted ones use defaults).

> **The asymmetry to remember:** for the *same* name, a **positional** call prefers a user function /
> struct over a built-in, but a **named** call prefers the **built-in**. This is why you can wrap or
> extend a built-in by name with a positional user function, yet `foo(x = 1)` still hits the built-in.

The selected resolution is **cached on the call site** and re-resolved whenever new functions, structs,
or module groups are defined.

### The complete error catalogue

| Situation | Message |
|-----------|---------|
| callee is not a plain identifier | `Can only call functions by name` |
| an argument evaluates to `nil` | `Passed parameter for function 'name' evaluated to nil.` |
| name unknown | `Unknown function 'name'` |
| right count, types don't fit | `Function 'name' has no overload matching types ({types})` |
| no compatible count | `Function 'name' has no overload that takes {n} argument(s)` |
| named: no matching overload | `Function 'name' has no overload matching named parameters ({names})` |
| a required param is missing | `Missing required parameter 'p' when calling 'name'` |
| built-in named call, unknown param | `Function 'name' has no parameter named 'p'` |
| built-in named call, wrong count | `Function 'name' expects {n} argument(s), but got {m}` |
| built-in has no named params | `Function 'name' does not support named parameters` |
| struct positional args | `Struct 'Name' instantiation with positional arguments is not supported. …` |
| struct unknown field | `Struct 'Name' has no field 'f'` |
| unknown struct | `Unknown struct 'Name'` |

---

## Calling

### Positional

```jask
add(2, 3)
```

### Named arguments

Call with `param = value` in **any order**; the interpreter reorders them to the declaration order:

```jask
function named(p: number, q: number)
    return p * 100 + q
endfunction

named(q = 2, p = 3)   ; 302
```

- For **built-in** functions, named calls must supply **exactly** the declared parameters
  (`Function 'name' has no parameter named 'x'` / `expects N argument(s), but got M`).
- For **user** functions, omitted parameters fall back to defaults.

### Passing arguments

- Arguments are evaluated **before** the call.
- An argument that evaluates to **`nil` is an error**:
  `Passed parameter for function 'name' evaluated to nil.` (jask does not pass `nil` to functions.)
- You can only call a function **by name** (the callee must be a plain identifier).

---

## Return values

```jask
function classify(n: number)
    if n > 0
        return "positive"
    endif
    if n < 0
        return "negative"
    endif
    return "zero"
endfunction
```

- `return` may carry an expression, or be bare (`return`) → returns `nil`.
- A function that reaches its end without a `return` returns **`nil`**.
- `return` exits the whole function, even from nested `if`/`while`/`for` (see
  [Control Flow → return](04-control-flow.md#return)).
- Only usable inside a function: `'return' is only allowed inside a function`.

---

## Function visibility & semantics

- Functions live in the current **module's** function table (the global table for the main script).
- A function is available **from the point its `function` statement executes** — jask does **not**
  hoist, so define before you call in source order:

  ```jask
  printLine(foo())        ; Unknown function 'foo'  (foo not defined yet)
  function foo() return 42 endfunction
  ```

- Functions are **not first-class values**: jask has no closures, anonymous functions, or the ability
  to assign a function to a variable or pass it as an argument. Functions are referenced purely by
  name.

---

## Next

- [Structs](07-structs.md) — the other user-defined type.
- [Modules](09-modules.md) — calling functions across modules with `mod::fn(...)`.
- [Built-in Functions](11-built-in-functions.md) — the interpreter's own functions (also match the call rules above).
