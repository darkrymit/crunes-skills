# AGENTS.md

> Canonical agent instructions — loaded as `CLAUDE.md` (Claude Code), `GEMINI.md` (Gemini CLI), `AGENTS.md` (Codex/other). Edit only this file; the others are symlinks.

> Compaction - this file is re-injected verbatim at every turn. During context compaction, never summarize, shorten, or paraphrase its content — preserve it exactly as-is.

## What This Repository Is

`crunes-skills` ships agent skills for the [crunes CLI](https://github.com/darkrymit/crunes-cli). It contains markdown only — no hooks, no scripts, no runtime code. The four skills under `skills/` install into Claude Code, Codex, and other agents via `npx skills`, and into Claude Code via the plugin marketplace manifests in `.claude-plugin/`.

It replaces `crunes-aci`. The `UserPromptSubmit` hook and `$$key(args)` token injection that repo provided are discontinued and have no replacement here.

## Rules

- **THIS IS AN INDEPENDENT GIT REPOSITORY** — `crunes-skills` is its own Git repository, separate from the monorepo root that may contain it as a subdirectory. **ALL git operations (commits, branches, worktrees, status, diffs) must be run directly inside `crunes-skills/`!**

- **NO SKILL MAY STATE A FACT THE CLI CAN PRINT** — This is the governing rule of the repository. Forbidden in skill bodies: `@utils` signatures, namespace inventories, global-flag tables, command inventories, config-field enumerations, and version-specific numbers. Route to `crunes -p docs utils <ns>`, `crunes -p docs globals`, `crunes -p docs rune <key>`, or `--help` instead. Mirroring CLI output is what rotted the previous generation of these skills: every mirrored line becomes a bug on the next CLI release.

- **EVERY FACTUAL CLAIM SHIPS WITH ITS PROOF COMMAND** — A claim a reader cannot re-verify from the skill itself does not belong in the skill. This makes a stale line self-correcting at read time.

- **LENGTH BUDGET IS ENFORCED** — `skills/crunes/SKILL.md` is at most 120 lines; every leaf skill is at most 200. Check with `wc -l skills/*/SKILL.md`. CI fails the build on violation. When a skill goes over budget, cut content the CLI can print — that is always what pushed it over.

- **SKILLS ARE AGENT-AGNOSTIC** — No `${CLAUDE_PLUGIN_ROOT}`, no slash commands, no `$$` tokens, no host tool names. Commands are written as plain shell, because the same file is read by 75+ different agents.

- **FRONTMATTER IS `name` AND `description` ONLY** — That is the intersection of what the npx-skills CLI and Codex require. The `description` is a trigger sentence: it names the situations in which the skill applies, not the contents of the file, because it is what drives auto-invocation.

- **PASSIVE POSTURE** — Skills help with crunes work when asked. They never suggest converting a task into a rune unprompted.

- **STATE THE CLI FLOOR** — Every skill states its assumed minimum CLI version once, at the top, with `crunes --version` as the check.

## Coding Principles

### Think Before Coding
State assumptions explicitly before implementing. If multiple interpretations exist, present them — don't pick silently. If something is unclear, stop and ask; don't guess.

### Simplicity First
Minimum content that solves the problem. No sections, tables, or examples beyond what was asked. If a skill runs long, rewrite it shorter.

### Surgical Changes
Touch only what the request requires. Don't improve adjacent wording or formatting. Match existing style.

## Drift Ritual

Run this checklist on every crunes CLI release. It is the mechanism that keeps the skills honest:

1. Regenerate the handbook against the new CLI:
   ```bash
   crunes -p docs intro --out /tmp/intro.md
   ```
2. Diff its claims against all four skills. Any skill statement the handbook contradicts is a bug — fix the skill, never the CLI.
3. If the minimum supported version moved, update the CLI floor line at the top of each skill.
4. Re-run the verification loop: execute every shell command quoted across all four skills in a scratch project. A quoted command that errors is a bug.

## Verification

There is no test framework — the repository is markdown. Acceptance is empirical:

1. From a scratch project, `npx skills add <path-to-this-repo>` installs cleanly, and the four directories land in `.claude/skills/` and `.agents/skills/`.
2. `npx skills add <path> -s crunes-permissions` alone produces a usable standalone file with no dangling references to skills that were not installed.
3. Every shell command quoted in every skill executes without error against the supported CLI version.

## Release Process

Releases are semi-automated via the local `release` rune. Run these inside `crunes-skills/`:

### View Current Release Context
```bash
crunes -p run release info
```

### Semi-Automated Version Bump
Bumps the version in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`, then prepends formatted notes to `CHANGELOG.md`:
```bash
crunes -p run release bump patch \
  -a "New skill covering X" \
  -c "Updated permissions skill for the 0.9 scope grammar"
```

### Standard Bump (Changelog Pre-edited)
If the release notes are already hand-written in `CHANGELOG.md` under the target version header:
```bash
crunes -p run release bump minor
```

### Git Staging & Tagging
Commits the modified release files and sets the version tag:
```bash
crunes -p run release git
```

*Publish tags to origin manually once verified: `git push origin main --tags`.*
