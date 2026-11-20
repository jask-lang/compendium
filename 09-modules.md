# Modules

[← Back to the Compendium index](README.md) · Prev: [Error Handling](08-error-handling.md) · Next: [Security and Permissions](10-security-and-permissions.md)

Modules let you package code and import it by name. The standard library **jcore** is distributed as
modules. Source of truth: the `Statement.Use` case and `ResolveModulePath` in
[`Interpreter.cs`](../interpreter/Interpreter.cs).

---

## Importing

```jask
use "jcore/math" as math
printLine(math::max(1, 2))     ; 2
```

- `use <path> as <alias>` — the path is a string expression; the alias must start with a **lowercase**
  letter.
- A `.jask` extension is appended automatically if the path doesn't already end in `.jask`
  (`use "mathlib" as m` loads `mathlib.jask`).
- The **same alias** can't be reused: `Module alias 'math' is already in use`.
- Once imported, the module's **exported** functions and structs are accessed through the alias with a
  double colon: `math::max(...)`, `date::Date(...)`.

---

## What a module is

Importing a module runs it in a **separate interpreter** that has its own:

- variables (module-private state),
- functions,
- struct types,
- sub-module imports.

It **shares** the same permission set as the importing program. A module's non-exported names are
invisible to importers — only `export`ed items cross the boundary.

Modules can import other modules (e.g. `jcore/date.jask` starts with `use "jcore/string" as str`).

---

## Exports

Only items marked `export` are visible to importers:

```jask
export function isPrime(n: number)
    ; ...
endfunction

function _helper(n: number)      ; private to this module
    ; ...
endfunction

export struct Date
    set day = 1
    set month = 1
    set year = 1970
endstruct
```

- Calling a non-exported function through the alias is an error:
  `Function 'helper' is not exported by module 'm'`. Same for structs:
  `Struct 'Foo' is not exported by module 'm'`.
- Within the module itself, private names work normally.
- By convention, private helpers are prefixed with `_`.

---

## Calling into a module

```jask
use "jcore/math" as math
use "jcore/date.jask" as date

math::max(1, 2)                       ; positional
math::power(base = 2, exponent = 4)   ; named arguments
date::Date(year = 2026, month = 1, day = 31)   ; instantiate an exported struct
```

- Arguments are evaluated in the **caller's** scope, then handed to the module's interpreter.
- An unknown alias is an error:
  `Unknown module 'math'. Did you forget a 'use ... as math' statement?`.
- Use [`unfoldModule(alias)`](11-built-in-functions.md#module-functions) to inspect a loaded module's
  exported functions/structs at runtime.

---

## Embedded vs file-system modules

jcore modules (`jcore/…`) are **embedded in the interpreter executable** — they are loaded from the
assembly's resources and do **not** require the `read` permission.

Any other module is loaded **from the file system** and **does** require the `read` permission:
`Missing permission 'read' for loading modules` (no read at all) or
`Missing permission 'read' on '<path>' for loading module` (path not permitted).

### Module path resolution

For a file-system module, the path is resolved in this order:

1. if it is an **absolute path** and the file exists → use it;
2. otherwise, relative to the **importing script's directory**;
3. otherwise, relative to the **process start directory**;
4. if not found anywhere, the importing-script-relative path is reported:
   `Module at '<path>' could not be found`.

> So `use "./helpers/util.jask"` resolves relative to the directory of the file containing the `use`,
> and a bare `use "util.jask"` is found next to that file.

### Circular imports

A module that (transitively) `use`s itself is detected:
`Circular 'use' detected: module '<path>' is already being loaded`.

---

## The special `jcore/http` module

`use "jcore/http" as http` loads the interpreter's built-in HTTP functions
([`get`](11-built-in-functions.md#http-functions) / [`post`](11-built-in-functions.md#http-functions)),
available only through that module alias and gated by the `allow-network` permission.

---

## Next

- [Security and Permissions](10-security-and-permissions.md) — how module loading interacts with permissions.
- [jcore Standard Library](12-jcore-standard-library.md) — the built-in modules in detail.
