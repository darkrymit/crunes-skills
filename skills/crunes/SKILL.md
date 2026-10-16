---
name: crunes
description: Use when working in a project that has a .crunes directory, when asked to run or list runes, when deciding whether crunes is the right tool for a task, or when you need to know which crunes skill to load next.
---

# crunes

Runes are project-local JavaScript modules that execute in a sandbox and return structured output. You reach them through the `crunes` CLI. Assumes `crunes >= 0.10.0` — check with `crunes --version`.

## Orient

```bash
crunes --version    # is the CLI installed at all
crunes -p doctor    # Node version, config validity, rune count, global config
crunes -p list      # what runes exist here; the Layer column shows project vs global
```

Use `-p` for every call. It strips colors and box-drawing, which is what you want when reading output.

In a directory with **no** `.crunes/config.json`, crunes runs *rootless*: only the global config layer applies, and all local state is redirected under `~/.crunes/rootless/<hash>/`, leaving the directory byte-identical. `crunes init` turns such a directory into a project.

## Run a rune

```bash
crunes [global-flags] run [run-flags] <key>[-s section] [rune-args...]
```

Four things the parser will not forgive:

1. **Global flags go before `run`.** `crunes --cwd . -p run demo` works; `crunes run --cwd . demo` fails with `Unknown option or misplaced flag`.
2. **Section filters live only inside the key brackets.** `demo[-s example-md]` works. Before the key it is rejected as misplaced; *after* the key it is silently handed to the rune as a raw argument.
3. **`--help` is not intercepted.** If a rune does not declare its own help option and check it, `crunes run demo --help` **executes the rune for real**. To inspect a rune without running it, use `crunes -p docs rune <key>`.
4. **An unmatched subcommand does not error.** It falls through to the root with the unknown token in `args.$rest`.

Verify any of these against the real CLI with `crunes -p run --help`.

### Batching

`-b` alone is not enough — batching is permission-gated per rune:

```bash
crunes -p run -b "demo[-s example-md]" + other-rune arg
```

Without a `batch` block in the rune's config entry this fails with `Batch not permitted for "<key>"`. The fix is a `"batch": { "allow": ["*"] }` entry alongside the rune's `path`; patterns match the argument string after the key, and `deny` is checked first. See `crunes-permissions`.

### Key prefixes

| Form | Resolves |
|---|---|
| `demo` | project config first, then enabled plugins, then global |
| `project:demo` | project config only |
| `plugin:demo` | a specific enabled plugin |
| `marketplace@plugin:demo` | fully qualified — required when two enabled plugins share a bare name |

An ambiguity error always lists the full keys to use. `crunes -p docs rune <key>` accepts the same forms.

## Choose the execution mode

- **`run`** — one-shot. The default; use it unless something below applies.
- **`repl`** — `crunes -p repl <key>` keeps the isolate alive across inputs, so state persists between turns. Needs its own `repl` permission block, which does not inherit from `run`.
- **background job** — for long-running work. `crunes -p job list`, `crunes -p job kill <id>`.
- **global rune** (`-g`) — lives in `~/.crunes/` and is callable from any directory. Read `crunes-permissions` before adding one: a global grant applies in *every* directory you invoke it from.
- **plugin rune vs local rune** — keep it local when the logic is specific to this project; package a plugin when it is not.

Check cost before wiring a rune into a loop: `crunes -p bench <key>` reports per-rune time and a fast/ok/slow verdict.

## The CLI is the documentation

Prefer these over recalling anything from memory — they are generated from the installed version, so they cannot be stale:

```bash
crunes -p docs rune <key>       # a rune's args schema, commands, examples, batch status
crunes -p docs utils            # available @utils namespaces
crunes -p docs utils <ns>       # full signatures for one namespace
crunes -p docs globals          # sandbox globals available without import
crunes -p docs intro            # the full workspace handbook
```

## Which skill next

- Writing or editing a rune → **`crunes-write-rune`**
- A permission denial, or deciding what belongs in an allow list → **`crunes-permissions`**
- Building or publishing a plugin → **`crunes-write-plugin`**
