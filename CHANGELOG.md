# Changelog

## [0.10.1] - 2026-11-21

### Removed
- **Config migration reference**: Drop `crunes migrate` mention from `crunes-permissions`

## [0.10.0] - 2026-11-21

### Added
- **Script runes & dual runtime tiers**: Document isolate runes (`*.rune.js`) vs native script runes (`bash`, `python`, `node`, CLI programs), interpreter resolution, `CRUNES_*` env vars, `args` schemas, and output modes in `crunes-write-rune`
- **Wildcard isolate permission grant**: Document `"allow": ["*"]` with `deny` precedence in `crunes-permissions`

### Changed
- **Config v2 (`run` / `repl` blocks)**: Update all four skills for `run` and `repl` config blocks (replacing top-level `path` / `permissions`), `batch` sibling positioning, and `crunes >= 0.10.0` CLI floor
- **Plugin manifest alignment**: Align `crunes-write-plugin` to manifest format `"1"`
- **Line-budget compliance**: Trim `crunes-write-rune/SKILL.md` under the 200-line CI limit

## [0.1.0] - 2026-08-11

### Added
- Four agent skills: `crunes`, `crunes-write-rune`, `crunes-permissions`, `crunes-write-plugin`.
- Installable via `npx skills add darkrymit/crunes-skills` and the Claude plugin marketplace.

### Changed
- Replaces `crunes-aci`. The `UserPromptSubmit` hook and `$$key` token injection are not carried forward.
