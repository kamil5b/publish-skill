# Privacy

`publish-whats-new` collects no data and has no server or telemetry.

It runs locally inside Claude Code and only does what you confirm:

- Reads your repo's git history and diff, on your machine.
- Writes release notes and, if you enable it, a version bump commit, in your repo.
- Calls `git push` and the `gh` CLI (GitHub releases) with your own credentials.

Your repo content is processed by Claude as part of your normal Claude Code session, under your Anthropic terms and privacy policy. This plugin adds no other data sharing.

Questions: open an issue at https://github.com/kamil5b/publish-skill/issues.
