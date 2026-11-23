---
name: publish-whats-new
description: Publish a release with a "What's new" changelog, driven by per-repo config. Tag style (vX.YY.ZZ, vX.Y.Z, timestamp, custom), audiences (non-tech what's new, tech notes, developer notes) and publish target are all configurable. Use when user says "publish the release", "publish what's new", "release vX.Y.Z", or "cut a release".
---

# Publish What's New

Tag latest origin branch tip, write release notes per user config, publish to GitHub.

Config file: `.claude/publish-whats-new.json` in the target repo. Schema and presets: `config-reference.md` (next to this file).

## Step 0 — Load config

1. Read `.claude/publish-whats-new.json`.
2. **Missing:** run setup. Ask user (AskUserQuestion, one batch) for:
   - Tag style: auto (default: detect from repo tags, fallback vX.Y.Z) / `vX.Y.Z` / `vX.YY.ZZ` / timestamp / custom regex
   - Sections: non-tech "What's new", tech notes, developer notes (multi-select)
   - Publish target: GitHub release / tag only / notes only (print)
   - Excludes: files/paths to leave out of the diff (globs or regexes) and commit subjects to ignore (regex). Offer quick picks inferred from the repo (look at its top-level dirs and files), e.g. docs (`*.md`, `docs/`), tests (`*.test.*`, `*.spec.*`, `tests/`), lockfiles, generated code, snapshots, `chore|ci|test` commits. Only suggest what exists in this repo. Multi-select plus "Other" for custom globs/regexes. "None" is valid.
   Sort answers: path globs → `diff.exclude`, path regexes → `diff.excludeRegex`, commit regexes → `log.excludeRegex`. Write config from answers using presets in `config-reference.md`, show it, then continue. Config is a repo file: tell user to commit it. Do not commit for them.
3. **Present:** validate against schema. Unknown/invalid field → stop, report which.
4. Per-run override: if user's request states a version, section choice, etc., it beats config for this run only.

## Step 1 — Resolve tag style

Always first: `git fetch <remote> --tags --prune`.

`tag.style` (default `auto`):

| style | Pattern |
|-------|---------|
| `auto` | Detected from repo history (below). |
| `semver` | `^v\d+\.\d+\.\d+$` (vX.Y.Z) |
| `semver-padded` | `^v\d+\.\d{2}\.\d{2}$` (vX.YY.ZZ) |
| `timestamp` | `^v\d{4}\.\d{2}\.\d{2}-\d{4}$`, generated from `tag.format` |
| `custom` | `tag.pattern` regex from config |

### Auto detection

1. List tags newest first: `git tag -l --sort=-v:refname --sort=-creatordate | head -20`. Last `--sort` is primary: date first, version order breaks same-second ties.
2. Classify each tag against known styles: `semver-padded`, `semver`, `timestamp`. Optional `v` prefix is detected too (`1.2.3` vs `v1.2.3`). Tags matching none are ignored (not releases).
   - `semver-padded` only if minor and patch are two-digit and at least one has a leading zero (e.g. `v1.02.03`). Otherwise `semver`. `v1.10.11` alone is `semver`.
3. Winner = style with most matching tags. Tie → style of the most recent matching tag.
4. **No tags, or none match any style → fall back to `semver` (vX.Y.Z), starting at `v0.1.0` unless user gives a version.**
5. Report detected style and the evidence (e.g. "auto: semver-padded, 8 of 8 recent tags"). If detection is mixed (any non-winning matches), mention it.

Detected style is used for validation, sorting and bumping this run. It is not written to config.

`tag.sort` (optional override): defaults to `-v:refname` for semver styles and `-creatordate` for `timestamp`/`custom`. When using `-creatordate`, always add `--sort=-v:refname` before it as tie-break (`git tag -l --sort=-v:refname --sort=-creatordate`).

## Step 2 — Resolve version and previous tag

Version source, `tag.mode` (default: `manual` if user gave a version, else `bump` for semver styles, `auto-generate` for timestamp):

- `manual`: version from user request. Must match style pattern. No match → stop, show an example in detected style, ask.
- `bump`: next from latest tag. `tag.bump` = `major|minor|patch` (default `patch`; if diff shows breaking changes, suggest `major`/`minor` and ask). Preserve padding.
- `auto-generate`: from `tag.format` (default `v%Y.%m.%d-%H%M`) using UTC or `tag.timezone`, via `date`.

Previous tag: latest tag matching style pattern, ordered by `tag.sort`.

- New tag already exists → stop, report.
- Version-sorted style and new version <= latest → stop and ask.
- No previous tag → first release. Log range is the whole history (`<remote>/<baseBranch>`, no `..`). Diff `<from>` is the empty tree (`git hash-object -t tree /dev/null`), so the root commit is included. Do not use the root commit as `<from>`: that drops it.

## Step 3 — Collect changes

Range: `<prev-tag>..<remote>/<baseBranch>`, or the whole history on a first release (see Step 2).

```bash
git log --no-merges --pretty='%h %s' <range>
```

- Drop log lines whose subject matches any `log.excludeRegex` (ERE, case-insensitive), e.g. `^(chore|ci)(\(.+\))?:`. Filter with `grep -viE`.
- Empty log (after filtering) → stop, report "nothing to release".
- Diff: if `diff.command` set, run it (substitute `{from}`, `{to}`, `{out}`). Else build the file list, then diff only those files:
  ```bash
  git diff --name-only -z <from>..<to> -- . ':!<glob1>' ':!<glob2>' \
    | grep -zvE '<regex1>|<regex2>' \
    | xargs -0 git diff <from>..<to> -- > <scratchpad>/release.diff
  ```
  - `diff.exclude`: glob pathspecs (`':!*.md'`).
  - `diff.excludeRegex`: ERE regexes matched against the full repo-relative path (e.g. `^docs/`, `\.generated\.`, `(^|/)__snapshots__/`). A file matching any glob or regex is excluded.
  - Keep `-z` / `grep -z` / `xargs -0` so names with spaces work. Drop the `grep` stage when `excludeRegex` is empty. If the file list is empty, `xargs` would diff everything: check first (`... | head -c1`) and skip.
  - Empty file list → note "no included file changes", still use the log.
  - Always write to scratchpad, never repo root.
- Excluded paths (docs/tests) will not be in diff; use log to catch them.
- Read diff for real changes. Log gives hashes and messages.

## Step 4 — Write notes

Write to scratchpad (never repo): `release-notes-<version>.md`.

Emit only sections with `enabled: true`, in config order, using each section's `title`. Built-in section kinds:

| kind | Audience | Content rules |
|------|----------|---------------|
| `whats-new` | Non-technical | Plain-language benefit per line. No file/function names, jargon, hashes. Say what users can now do or what got fixed. Group related commits. Skip refactors, chores, tests. |
| `tech-summary` | Technical users (integrators, power users, QA) | Behavior/API/UX changes, config users must touch, known limits. Light jargon OK. Hashes optional. |
| `developer-notes` | Engineers | Commit hashes, migrations, schema, API, env vars, dependencies, breaking changes. Use `subsections` from config; default: Changes, Migrations/config/env, Breaking changes, Dependencies. Write "None" for empty subsections. |
| `custom` | Anything | Follow section's `prompt` field verbatim. |

Rules for all:
- Only claim what diff/log supports. No invented features.
- Section's `style` (`bullets` default | `prose` | `table`) and `maxItems` honored.
- `notes.language` sets output language (default: language of user's request).
- If `notes.header` / `notes.footer` set, add verbatim.
- Section enabled but nothing relevant → omit it, mention in report (never pad).

If only `whats-new` enabled, no developer content may leak in (no hashes, no file names).

## Step 5 — Confirm (if configured)

`publish.confirm: true` (default) → show tag, previous tag, commit count and the notes, ask go/no-go before publishing. `false` → user's request is the go-ahead; proceed.

## Step 6 — Publish

By `publish.target`:

- `github-release`:
  ```bash
  gh release create <version> --target <baseBranch> --title "<title>" --notes-file <notes-file> [--draft] [--prerelease] [--latest=false]
  ```
  Title = `publish.title` template (`{version}` default). Creates tag on remote at branch tip; do not push a separate tag.
- `tag-only`: `git tag -a <version> <remote>/<baseBranch> -m "<one-line>"` then `git push <remote> <version>`. Notes printed, not stored.
- `notes-only`: print notes, publish nothing, create no tag.
- `file`: write notes to `publish.file` path (supports `{version}`), no tag. Tell user; do not commit.

## Step 7 — Verify and report

`gh release view <version>` (github-release) or `git ls-remote --tags <remote> <version>` (tag-only). Report: URL/tag, previous tag, commit count, and the user-facing section(s) of the notes.

## Guardrails

- Never force-push, delete, or move existing tags.
- Never edit repo files or commit, except writing the config on first-run setup (and only that file).
- Tag origin/remote branch tip, not local HEAD.
- `gh` unauthenticated → tell user to run `! gh auth login`.
- Config disagrees with user's explicit request → follow request, note the deviation.
