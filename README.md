# publish-skill

Claude Code skill: publish a release with a "What's new" changelog.

- Tag style configurable: auto (default, detects from past tags, falls back to `vX.Y.Z`), `vX.YY.ZZ`, `vX.Y.Z`, timestamp, or any regex.
- Audience configurable: non-tech "What's new", tech summary, developer notes, or custom sections.
- Target configurable: GitHub release, tag only, notes only, or file.

## Install

Plugin:

```
/plugin marketplace add kamil5b/publish-skill
/plugin install publish-whats-new@publish-skill
```

Local clone works too: `/plugin marketplace add /path/to/publish-skill`.

Or copy `skills/publish-whats-new/` to `~/.claude/skills/` (global) or `<repo>/.claude/skills/` (per repo).

## Configure

First run asks questions and writes `.claude/publish-whats-new.json`. See `skills/publish-whats-new/config-reference.md`.

## License

MIT. See `LICENSE`.
