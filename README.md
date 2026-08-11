# crunes-skills

Agent skills for [crunes](https://github.com/darkrymit/crunes) — small, project-local scripts (*runes*) that AI coding agents run through a sandboxed CLI to gather live project knowledge or perform structured actions.

This repository contains skills only: markdown files that teach an agent how to use the `crunes` CLI well. It is not the CLI, and it ships no hooks or runtime code.

## Prerequisites

All rune logic lives in the CLI. Install it first:

```bash
npm install -g @darkrymit/crunes-cli
crunes --version
```

Requires `crunes >= 0.8.5`.

## Installation

Install into whichever agents you use — the skills CLI detects them and routes files to the right directories:

```bash
npx skills add darkrymit/crunes-skills
```

Install a single skill:

```bash
npx skills add darkrymit/crunes-skills -s crunes-permissions
```

Claude Code users can also install through the plugin marketplace:

```bash
/plugin marketplace add darkrymit/crunes-skills
/plugin install crunes-skills
```

Each skill is standalone — installing one alone works fine.

## Skills

| Skill | Use when |
|---|---|
| `crunes` | Orienting in a project with a `.crunes` directory, listing and running runes, deciding whether crunes fits the task, choosing which skill to load next |
| `crunes-write-rune` | Creating or editing a rune, choosing what it returns, debugging how its arguments parse |
| `crunes-permissions` | Hitting a permission denial, writing an allow list, granting a plugin rune access, scoping a global rune |
| `crunes-write-plugin` | Packaging runes into a plugin, writing a manifest, publishing for others |

## Design

These skills deliberately contain no `@utils` signatures, namespace lists, flag tables, or command inventories. The CLI generates all of that from the installed version:

```bash
crunes -p docs rune <key>       # a rune's args schema, commands, examples
crunes -p docs utils <ns>       # full signatures for one namespace
crunes -p docs globals          # sandbox globals available without import
crunes -p docs intro            # the full workspace handbook
```

Skills carry what the CLI cannot print: judgment, ordering, invariants, and the gotchas that cost an agent a wasted run. Every factual claim ships with the command that re-proves it, so a stale line corrects itself at read time.

## Replaces crunes-aci

This repository supersedes [crunes-aci](https://github.com/darkrymit/crunes-aci), which was Claude Code-only. Its `UserPromptSubmit` hook and `$$key(args)` prompt-token injection are discontinued and have no replacement — agents call the CLI directly instead, which works identically in a terminal, in CI, and in any agent.

## License

MIT — [Tamerlan Hurbanov (DarkRymit)](https://github.com/darkrymit)
