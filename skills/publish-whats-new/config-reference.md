# publish-whats-new config reference

File: `.claude/publish-whats-new.json` (in the repo being released).

## Full example

Values are illustrative. Excludes default to empty; first-run setup asks what to exclude.

```json
{
  "remote": "origin",
  "baseBranch": "main",
  "tag": {
    "style": "auto"
  },
  "diff": {
    "command": null,
    "exclude": ["*.md", "*.csv", "*.test.*", "*.spec.*"],
    "excludeRegex": ["^docs/", "\\.generated\\."]
  },
  "log": {
    "excludeRegex": ["^(chore|ci|test)(\\(.+\\))?:"]
  },
  "notes": {
    "language": null,
    "header": null,
    "footer": null,
    "sections": [
      { "kind": "whats-new", "enabled": true, "title": "What's new", "style": "bullets" },
      { "kind": "tech-summary", "enabled": false, "title": "Technical summary" },
      {
        "kind": "developer-notes",
        "enabled": true,
        "title": "Developer notes",
        "subsections": ["Changes", "Migrations / config / env", "Breaking changes", "Dependencies"]
      }
    ]
  },
  "publish": {
    "target": "github-release",
    "confirm": true,
    "title": "{version}",
    "draft": false,
    "prerelease": false
  }
}
```

## Fields

| Field | Default | Notes |
|-------|---------|-------|
| `remote` | `origin` | |
| `baseBranch` | `main` | Branch tip that gets tagged. |
| `tag.style` | `auto` | `auto`, `semver` (vX.Y.Z), `semver-padded` (vX.YY.ZZ), `timestamp`, `custom`. `auto` inspects the last 20 tags, picks the most common known style (tie: most recent tag), falls back to `semver` if repo has no matching tags. |
| `tag.pattern` | – | ERE regex. Required for `custom`. |
| `tag.mode` | smart | `manual` (user gives version; default when given), `bump` (default for semver styles), `auto-generate` (default for timestamp). |
| `tag.bump` | `patch` | `bump` mode: `major` / `minor` / `patch`. |
| `tag.format` | `v%Y.%m.%d-%H%M` | `auto-generate` mode: `date` format string. |
| `tag.timezone` | UTC | `auto-generate` mode. |
| `tag.sort` | by style | `-v:refname` for semver styles, `-creatordate` for timestamp/custom (version order is the automatic tie-break). |
| `diff.command` | null | Custom diff script. Placeholders `{from}` `{to}` `{out}`. |
| `diff.exclude` | `[]` | Pathspec globs excluded from diff. |
| `diff.excludeRegex` | `[]` | ERE regexes on repo-relative file paths; matches are excluded from diff. Combined with `exclude`. |
| `log.excludeRegex` | `[]` | ERE regexes (case-insensitive) on commit subjects; matches are dropped from the changelog input. |
| `notes.language` | null | null = match user's request language. |
| `notes.sections[]` | whats-new + developer-notes | Order = output order. `kind`: `whats-new`, `tech-summary`, `developer-notes`, `custom` (needs `prompt`). Optional `style`, `maxItems`. |
| `publish.target` | `github-release` | `github-release`, `tag-only`, `notes-only`, `file` (needs `publish.file`). |
| `publish.confirm` | `true` | `false` = request is the go-ahead, no second prompt. |
| `publish.title` | `{version}` | Release title template. |

## Tag style presets

**Auto (default)**: `{ "style": "auto" }`. Detects from history, fallback `vX.Y.Z`.
**Plain semver**: `{ "style": "semver" }`
**Padded semver**: `{ "style": "semver-padded" }`
**Timestamp**: `{ "style": "timestamp", "format": "v%Y.%m.%d-%H%M" }`
**Anything else**: `{ "style": "custom", "pattern": "^rel-\\d+$", "mode": "manual" }`

## Audience presets

| Want | sections |
|------|----------|
| Non-tech only | `whats-new` |
| Tech users only | `tech-summary` |
| Non-tech + dev notes | `whats-new`, `developer-notes` |
| Everything | `whats-new`, `tech-summary`, `developer-notes` |
| No dev notes | omit `developer-notes` |
