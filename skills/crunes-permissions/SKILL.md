---
name: crunes-permissions
description: Use when a rune hits a permission denial, when writing or widening an allow list in .crunes/config.json, when granting a plugin rune extra access, or when deciding what a global rune should be allowed to do.
---

# Rune Permissions

Runes have no ambient access to anything. Every capability — reading a file, globbing, running a command, opening a socket — is a scope string declared in `.crunes/config.json`. Assumes `crunes >= 0.10.0` — check with `crunes --version`.

## Fix a denial in three steps

A denial looks like this:

```
[err] Rune "perm" failed:
'fs.glob:*.md' is not permitted.
```

**The message contains the exact scope string the sandbox asked for.** Paste it verbatim into the rune's allow list and re-run. Do not paraphrase it, and do not reach for a wider pattern first — a related scope will not cover it. `fs.read:./**` does *not* satisfy `fs.glob:*.md`; glob is its own action.

```json
{
  "runes": {
    "perm": {
      "run": {
        "command": ".crunes/runes/perm.rune.js",
        "permissions": { "allow": ["fs.glob:*.md", "shell.run:git *"] }
      }
    }
  }
}
```

If the denial is a `shell.run` and the scope string looks stranger than the command you wrote, the problem is usually parsing rather than scoping — see below.

## Scope grammar

The shape is `<namespace>.<action>:<target>`, and the target syntax differs per action:

```
fs.read:src/**
fs.write:dist/**
fs.glob:*.md
http.fetch:GET::https://api.github.com/*
shell.run:git *
sqlite.read:./state::db
```

Some targets carry two components separated by `::` — a method and a URL pattern for `http.fetch`, a location and a database name for `sqlite`, a cwd and a pattern for the long form of `fs.glob`. `*` is a wildcard throughout.

Never guess which scope a function needs. Every `@utils` function documents its own requirement:

```bash
crunes -p docs utils fs        # each entry states "Requires `fs.<action>:<target>` permission"
crunes -p docs utils <ns>
```

`fs` paths resolve from the project root — the directory containing `.crunes/`.

## Debugging a shell denial

`crunes shell explain` shows exactly how a command line decomposes and which scope each part needs:

```bash
crunes -p shell explain "git log --oneline | head -3 > out.txt"
```

```
command positions:
  @   0  git log --oneline
          needs shell.run:git log --oneline
  @  20  head -3
          needs shell.run:head -3

redirects:
  @  30  out.txt
          needs fs.write:out.txt
```

Every pipeline stage needs its own grant, and a redirect target needs an `fs.write` — not a `shell.run`. Run this before widening a shell scope; the answer is often that you granted one stage of three.

## Grants live in the block they apply to

Grants sit inside the `run` block, and inside the `repl` block for a rune with a REPL. **`repl` does not inherit from `run`.** A rune that reads the same file in both modes lists the scope twice:

```json
"run":  { "command": ".crunes/runes/db.rune.js", "permissions": { "allow": ["sqlite.read:./state::db"] } },
"repl": { "command": ".crunes/runes/db.rune.js", "permissions": { "allow": ["sqlite.read:./state::db", "sqlite.write:./state::db"] } }
```

`deny` is available alongside `allow` in either block and is checked first. A script rune — a `run` command that is not a `*.rune.js` module — runs natively with full access, so crunes refuses grants on it rather than imply a sandbox. An entry still written with `path` or `permissions.run` is refused; `crunes migrate` rewrites it.

## Batching is a separate gate

`crunes run -b` is refused with `Batch not permitted for "<key>"` unless the rune's config entry declares a `batch` block. It sits beside `run`, not inside it:

```json
"demo": {
  "run": ".crunes/runes/demo.rune.js",
  "batch": { "allow": ["*"], "deny": [] }
}
```

Patterns match the argument string after the key, so `"allow": ["info*"]` permits `release info` in a batch while refusing `release bump`. `deny` wins.

## Config layers

Three layers merge, lowest to highest, later winning per key:

```
1. ~/.crunes/config.json                    global, machine-wide
2. <project>/.crunes/config.json
3. <project>/.crunes/config.local.json      never committed
```

`crunes -p list` prints a Layer column so you can see which layer a rune came from. `CRUNES_NO_GLOBAL=1` drops the global layer entirely — set it in CI so a developer's personal config can never alter project behavior.

Because `plugins` is a boolean map rather than a list, a higher layer can switch off what a lower one enabled: `crunes plugin disable <name>` writes `false` into the project config and leaves the global enable intact.

## Granting a plugin rune extra access

Plugin runes have no `runes.<key>` entry of their own. To give one project-side vars or permissions, add an entry keyed by the fully-qualified name:

```json
{
  "plugins": { "my-market@my-plugin": true },
  "runes": {
    "my-market@my-plugin:deploy": {
      "vars": { "region": "us-east-1" },
      "run": { "permissions": { "allow": ["fs.read:src/**"] } }
    }
  }
}
```

This key must **always** carry the marketplace prefix. A bare `"my-plugin:deploy"` is not merely ambiguous — it makes the whole config unreadable:

```
[err] Config unreadable: config.json: runes["my-plugin:deploy"] is a plugin-rune override
— but "my-plugin" is missing the marketplace prefix. Use the full
"marketplace@plugin:deploy" form.
```

Plugin runes always hold read access to their own plugin directory. Everything else is declared in the plugin manifest and consented to at install time.

## Global and rootless runes

A permission granted to a global rune in `~/.crunes/config.json` applies in **every directory** from which that rune is invoked.

In rootless mode — any directory with no `.crunes/config.json` — the global layer is the only layer, and both `@project/` imports and permission targets still resolve against the current working directory. This is what makes a generic "serve this directory" rune work from anywhere, and it is also the reason a broad global grant such as `fs.write:./**` is genuinely dangerous: it applies to whatever directory the user happens to be standing in.

Scope global-rune permissions narrowly and deliberately. Prefer a project-local rune whenever the work is project-specific.

## When the config itself is suspect

```bash
crunes -p doctor    # Node version, config validity, rune count, global config presence
```
