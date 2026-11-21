# Getting Started

[← Back to the Compendium index](README.md) · Next: [Language Basics](02-language-basics.md)

jask is distributed as a C# console application. You need the [.NET SDK](https://dotnet.microsoft.com/download)
(the project targets `net10.0`) and the interpreter repository checked out.

> Throughout this compendium, commands are shown as run from inside the `interpreter/` directory
> (`jask-lang/interpreter`).

---

## Installation / build

```bash
cd interpreter
dotnet build          # produces bin/Debug/net10.0/jask.dll
```

You can run the built interpreter directly, or through `dotnet run`:

```bash
dotnet run --no-build -- --version
# jask lang interpreter 0.0.1 (commit <hash> build on <date>)
```

The `--version` line also prints the git commit hash and build time embedded in the assembly.

---

## Running jask

Two modes exist:

### 1. Interactive REPL

With no `--input`, jask starts the interactive read-eval-print loop.

```bash
dotnet run
```

The REPL:

- prints a version banner, then the prompt `>>>`;
- evaluates each submitted line / block and **prints the result of any top-level expression**
  (e.g. typing `1 + 1` prints `2`);
- supports **multi-line entry**: when you open a block (`if`, `while`, `for`, `function`,
  `struct`, `try`) the prompt changes to `...` (indented) and lines are buffered until the
  matching closing keyword (`endif`, `endwhile`, `endfor`, `endfunction`, `endstruct`, `endtry`)
  is entered, then the whole block is executed;
- has **command history**: the arrow keys navigate up to 100 previous entries, persisted to
  `~/.jask_history`;
- has basic **syntax highlighting** (strings, numbers, booleans, `nil`, and keywords are colored);
- `Ctrl+L` clears the screen; type `exit` to quit.

A missing `stdout` permission does **not** stop the REPL — it only prints a yellow warning:
`Warning: Missing permission for 'stdout'!`.

### 2. Running a script

```bash
dotnet run --allow-stdout --input "myScript.jask"
```

- `--input` / `-i` takes the path to a `.jask` file. The file must end in `.jask`, otherwise:
  `Input 'x.txt' is not a jask file.` A missing file gives `Input 'x.jask' cannot be found.`
- The script's **directory** becomes the interpreter's base directory, used to resolve relative
  `use` module paths (see [Modules](09-modules.md#module-path-resolution)).
- Expression statements in a script are **not** printed (unlike the REPL); use `print`/`printLine`.

Example (the jcore test runner, run from the `interpreter/` directory):

```bash
dotnet run --allow-stdout --allow-stdin --allow-read --allow-write \
    --input="../jcore/test/test_basics.jask"
```

---

## Command-line flags

| Flag | Short | Effect |
|------|-------|--------|
| `--help` | `-h` | Print usage and exit |
| `--version` | `-v` | Print the interpreter version and exit |
| `--input` | `-i` | Run a `.jask` file instead of the REPL |
| `--allow-stdout` | `-ao` | Allow `print` / `printLine` |
| `--allow-stdin` | `-ai` | Allow `readInput` |
| `--allow-trust` | `-at` | Allow `trust()` |
| `--allow-interpret` | `-ae` | Allow `interpret()` — running dynamically-constructed jask code |
| `--allow-read` | `-ar` | Allow reading files / loading modules (optionally restricted to paths) |
| `--allow-write` | `-aw` | Allow `writeFile` (optionally restricted to paths) |
| `--allow-network` | `-an` | Allow `get` / `post` *(supported, but not listed in `--help`)* |
| `--allow-all` | — | Grant **every** permission; overrides all other flags *(not listed in `--help`)* |

> **Default is deny.** With no `--allow-*` flags, the script runs with **no** permissions and any
> privileged built-in will fail. See [Security and Permissions](10-security-and-permissions.md).

`--allow-read` and `--allow-write` may be followed by one or more **paths** and may be repeated. A
bare flag (no paths) allows *all* paths; with paths, access is restricted to those paths (and their
sub-trees). See [path-based permissions](10-security-and-permissions.md#path-based-readwrite).

Both `--flag value` and `--flag=value` forms are accepted, and flag names are case-insensitive.

---

## Output and errors

- Program errors are written to **stderr** in red, prefixed `Error:` (interpreter errors) or
  `Unexpected error:` (unexpected .NET exceptions):

  ```
  Error: [myScript.jask:3] Unknown variable 'foo'.
  ```

  The `[path:line]` prefix (when present) points at the source location of the error.
- Successful output from `print`/`printLine` goes to **stdout** and requires `--allow-stdout`.

---

## Where things live

```
jask-lang/
├── interpreter/      # the C# interpreter (this program)
│   ├── test/         # interpreter-level .jask tests
├── jcore/            # the standard library, written in jask
│   ├── math.jask  string.jask  date.jask
│   └── test/       # jcore test suite (run_all_tests.jask)
├── tutorials/        # step-by-step lessons
└── compendium/       # this reference
```
