# Prompt Logger

A Claude Code skill that installs a `UserPromptSubmit` hook logging every prompt
you submit — across every repo, on one machine — to a single centralized file:
`~/.claude/prompt-log.jsonl`.

Each line records a timestamp, repo name, machine name, session ID, and the
prompt text (with auto-injected context like `<ide_selection>` blocks stripped
out, so it's a record of what you actually typed).

An optional second script converts that JSONL into human-readable Markdown —
newest entry first — at any location you choose, such as a note in an Obsidian
vault. It can run on demand or on a 5-minute `launchd` schedule (macOS).

See [SKILL.md](SKILL.md) for the full install, verify, export, auto-sync, and
uninstall instructions.

## Why

If you use Claude Code across many repos and machines, there's no built-in way
to see everything you've asked it over time. This gives you one append-only
log you can grep, sync to notes, or just keep as an audit trail.

## Requirements

- macOS or Linux
- [`jq`](https://jqlang.org/) (`brew install jq` / `apt install jq`)

## License

Apache License 2.0 — see [LICENSE](LICENSE).
