---
name: crunes-write-rune
description: Use when creating a new rune, editing an existing one, scaffolding with crunes create or crunes template apply, choosing what a rune should return, or debugging how a rune's arguments are parsed.
---

# Writing Runes

A rune is an ESM module that executes inside an `isolated-vm` sandbox. There is no access to Node builtins — all I/O goes through `@utils`, and every capability must be granted in `.crunes/config.json`. Assumes `crunes >= 0.10.0` — check with `crunes --version`.

## Scaffold

```bash
crunes -p -y create <key> --format markdown     # --format is required non-interactively
crunes -p template list
crunes -p template apply <template> --as <key>
```

`create` writes `.crunes/runes/<key>.rune.js` and registers it in `.crunes/config.json` as the entry's `run` block for you. Use `-g` to place the rune in `~/.crunes/` instead, making it callable from any directory.

## Imports

Four rules, and they are absolute:

- `@utils` — always available.
- `./relative` — always available, resolved against the rune file.
- `@project/path` — the project root; requires an `fs.read` permission covering that path.
- `@plugin/path` — plugin runes only; resolves from the plugin's own root.
- `node:fs`, `node:path`, and every other `node:*` — **always blocked.** Use the `@utils` equivalent.

## Exports

Run mode uses `args`, `run`, and optionally `dispose`. REPL mode uses `argsRepl`, `repl`, `bannerRepl`, `commandsRepl`, `inputRepl`, `completeInputRepl`, and `disposeRepl`. For the full contract of any one of them:

```bash
crunes -p docs run          # run(args) and what args contains
crunes -p docs args         # args(builder)
crunes -p docs repl         # the repl lifecycle
crunes -p docs args-repl    # argsRepl(builder)
```

A minimal run-mode rune with a nested command:

```js
import { rune, section, md } from '@utils'

export function args(builder) {
  return builder
    .option('-h, --help', 'Show help')
    .option('--verbose', 'Verbose output', false)
    .command('remote', 'Manage remotes', remote => {
      remote.command('add', 'Add a remote', add => {
        add.positional('<name>', 'Remote name')
           .positional('<url>', 'Remote URL')
      })
    })
}

export async function run(args) {
  if (args.help) return rune.helpSection()
  if (args.$command === 'remote add') {
    return section.create('result', { type: 'markdown', content: md.p(`${args.name} -> ${args.url}`) })
  }
}
```

## Parsed arguments

This is where most mistakes happen. Inside `run(args)`:

| Property | Contains |
|---|---|
| `args.$command` | Matched command path, space-separated. `''` at root. Always present. |
| `args.$commands` | The same path as an array. |
| `args._` | Data positionals with command tokens stripped, so `args._[0]` is the first user value after the matched command. |
| `args.$rest` | Positionals not mapped to any named parameter. |
| `args.$raw` | The original unparsed argument array. |
| `args.name` | A positional declared as `<name>` maps to its own key automatically. |

Two consequences worth internalising:

- **Unknown commands do not error.** They fall through to root, so detect them yourself with `args.$command === '' && args.$rest.length > 0`.
- **`--help` is not intercepted by the runner.** Declare `.option('-h, --help', ...)` and return `rune.helpSection()` at the top of `run`, or `crunes run <key> --help` will execute the rune for real. `rune.helpText()` returns the same content as a plain string.

If you omit `args()` entirely, positionals still arrive in `args._`.

## What to return

`run` may return a single section, an array of sections, a plain string, or nothing at all. Sections are built with `section.create(name, data, opts?)`, where `data` is either `{ type: 'markdown', content }` or `{ type: 'tree', root }`. `section.emit(section)` streams one out *before* the function returns — use it for progress on long work.

```js
return [
  section.create('summary', { type: 'markdown', content: md.p('Done') }),
  section.create('detail', { type: 'markdown', content: md.ul(['a', 'b']) }, { title: 'Detail' }),
]
```

Section names matter: they are what `key[-s name]` filters on, and what `section.match()` and `section.selected()` operate against. For the builders themselves, read `crunes -p docs utils section` and `crunes -p docs utils md`.

## REPL mode

The isolate stays alive across inputs, so module-level variables *are* your session state:

```js
import { section, md } from '@utils'

let count = 0

export async function repl() { return 'counter> ' }

export async function inputRepl(input) {
  if (input.type === 'eof' || input.type === 'interrupt') return { type: 'done' }
  if (input.type === 'line' && input.text.trim() === 'inc') {
    section.emit(section.create('result', { type: 'markdown', content: md.p(`count: ${++count}`) }))
  }
}
```

Three traps:

- `argsRepl` does **not** fall back to `args`. If it is absent, `repl(args)` receives an empty object.
- A rune has a REPL only when its config entry declares a `"repl"` block naming the module. Its grants live in that block and do **not** inherit from `"run"`, so a rune that reads files in both modes lists those scopes twice. See `crunes-permissions`.
- `inputRepl` receives typed events — `line`, `command`, `interrupt`, `eof`. Return `{ type: 'done' }` to end the session, `{ type: 'prompt', value }` to change the prompt, or nothing to continue.

## Test loop

```bash
crunes -p run <key>              # does it produce the sections you expect
crunes -p run <key> --help       # does your help path work
crunes -p docs rune <key>        # is the args schema what you declared
crunes -p bench <key>            # fast, ok, or slow
crunes -p repl <key>             # interactive session
```

If a rune fails on a capability rather than a bug, go to `crunes-permissions`.

## Deliberately not in this file

`@utils` function signatures and sandbox globals are omitted on purpose — they change per release, and the CLI prints the current set:

```bash
crunes -p docs utils            # namespaces available
crunes -p docs utils <ns>       # signatures for one namespace
crunes -p docs globals          # what exists without importing anything
```

Read those rather than recalling a signature. A namespace you remember may have been renamed, and one you have not heard of may now exist.
