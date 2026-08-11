---
name: crunes-write-plugin
description: Use when packaging runes into a crunes plugin, writing a plugin manifest, registering a marketplace source, or publishing a plugin for installation by others.
---

# Writing Plugins

A plugin packages runes and templates so other projects can install them. Assumes `crunes >= 0.8.5` — check with `crunes --version`.

Reach for a plugin only when the logic is *not* specific to one project. Anything that reads this repo's layout, its config, or its conventions belongs in `.crunes/runes/` as a local rune. A plugin earns its packaging when several projects would install the same thing.

## Scaffold

Do not hand-build the directory — the manifest format is versioned and the scaffolder tracks it:

```bash
crunes -p -y plugin create <name> --description "What it does"
```

That produces:

```
<name>/
  .crunes-plugin/plugin.json         rune and template declarations, permissions
  .crunes-plugin/marketplace.json    the marketplace entry for this plugin
  runes/example.js
  templates/example-template.js
  README.md  CHANGELOG.md
```

Note the directory is `.crunes-plugin/`, and that rune files are found by convention at `runes/<key>.js` — plugin manifests declare no `path` field.

## plugin.json

```json
{
  "format": "1",
  "runes": {
    "example": {
      "name": "Example Rune",
      "description": "Replace with your rune description",
      "permissions": {
        "run": { "allow": [], "deny": [] }
      }
    }
  },
  "templates": {
    "example-template": { "name": "Example Template", "description": "..." }
  }
}
```

The `permissions` block here is what the installing user is shown and consents to. Declare the narrowest set that works — a plugin asking for `fs.write:./**` is one most people should decline. Lifecycle keys behave exactly as in a project config: `repl` does not inherit from `run`. See `crunes-permissions` for scope grammar.

## Writing the runes themselves

Plugin runes are ordinary runes — same exports, same `@utils`, same sandbox. See `crunes-write-rune`. Two differences:

- `@plugin/path` resolves from the plugin's own root, and a plugin rune always holds read access to its own directory without declaring it.
- Everything outside that directory — project files, network, shell, environment — must be declared in `plugin.json` and consented to at install time.

## Distribute and install

A marketplace source is any URL or local path holding a `marketplace.json`:

```bash
crunes -p marketplace add ./my-plugin          # local path, for testing
crunes -p marketplace add <url>                # published source
crunes -p marketplace browse                   # every plugin from every configured source
crunes -p marketplace search <query>
crunes -p marketplace list                     # configured sources and their names
crunes -p marketplace update <name>            # refresh one source's cached data
```

`browse` and `search` return nothing until at least one source is configured. `update` takes the marketplace *name* shown by `marketplace list`, not the URL it was added with.

Installing takes the `<marketplace>@<plugin>` form:

```bash
crunes -p plugin install my-market@my-plugin
crunes -p plugin install -g my-market@my-plugin    # enable in every project
crunes -p plugin list
crunes -p plugin update <plugin>
crunes -p plugin disable <plugin>                  # opt this project out
crunes -p plugin uninstall <plugin>
```

Install registers the plugin globally and enables it for the current project, prompting for permission consent. On update, only new or escalated permissions re-prompt.

Installed runes appear in `crunes -p list` under a `plugin` layer with a `<plugin>:<key>` key:

```
demo-plugin:example   Example Rune   plugin: demo-plugin@demo-plugin   plugin
```

Run them with that key. If two enabled plugins share a bare name, use the fully-qualified `marketplace@plugin:rune` form; the ambiguity error lists the exact keys.

## Test before publishing

```bash
crunes -p -y marketplace add ./<name>
crunes -p -y plugin install <name>@<name>
crunes -p list
crunes -p docs rune <name>:<key>
crunes -p run <name>:<key>
```

Install from the local path first and exercise every rune. A permission you forgot to declare surfaces here as a denial, not in a user's project.

## Project-side configuration

A project can give an installed plugin rune its own vars or extra permissions, keyed by the fully-qualified `marketplace@plugin:rune` string — never a bare plugin name, which makes the config unreadable. `crunes-permissions` covers the exact form.
