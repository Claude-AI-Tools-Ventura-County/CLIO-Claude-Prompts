---
name: clio
description: Install, verify, or uninstall a Claude Code hook that logs every submitted prompt to a centralized JSONL file at ~/.claude/prompt-log.jsonl, tagged with timestamp, repo name, git branch, and machine name — and optionally export that log to a human-readable Markdown file at any location, such as an Obsidian vault. Use when the user asks to set up prompt logging, track/save Claude Code prompts, build a prompt history or audit trail, sync prompts into Obsidian/notes, or mentions wanting a centralized, readable record of what they've asked Claude Code across projects.
---

# CLIO

Installs a `UserPromptSubmit` hook (user-scope, applies to every repo) that appends
one JSON line per submitted prompt to a single centralized file:
`~/.claude/prompt-log.jsonl`.

A second, optional script converts that JSONL into a human-readable Markdown file
at any location you choose — an Obsidian vault, a notes folder, wherever. It's kept
separate from the hook on purpose: the hook must stay fast (it blocks every prompt
for up to 30s), so formatting happens later, on demand, not inline.

## What gets logged

One line per prompt:
```json
{"timestamp":"2026-07-09T18:42:11Z","repo":"hypercart","branch":"main","machine":"Noels-MacBook-Pro","session_id":"abc123","prompt":"..."}
```

## Install

Run once in a terminal (macOS/Linux). Requires `jq` — install it first if needed
(`brew install jq` on macOS, `apt install jq` on Debian/Ubuntu):

```bash
command -v jq >/dev/null 2>&1 || { echo "jq is required. Install it: brew install jq (macOS) / apt install jq (Linux)"; return 1 2>/dev/null || exit 1; }

mkdir -p ~/.claude/hooks

cat > ~/.claude/hooks/log-prompt.sh << 'EOF'
#!/bin/bash
# Logs every Claude Code prompt to a centralized JSONL file.
# Never blocks prompt submission (always exits 0) — failures go to a
# separate error log instead of silently dropping the prompt.
input=$(cat)
errlog="$HOME/.claude/prompt-log-errors.log"
ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)

if ! command -v jq >/dev/null 2>&1; then
  echo "$ts jq not found — prompt not logged" >> "$errlog"
  exit 0
fi

repo=$(basename "$(git -C "$CLAUDE_PROJECT_DIR" rev-parse --show-toplevel 2>/dev/null || echo "$CLAUDE_PROJECT_DIR")")
branch=$(git -C "$CLAUDE_PROJECT_DIR" rev-parse --abbrev-ref HEAD 2>/dev/null || echo "")
machine=$(scutil --get ComputerName 2>/dev/null || hostname -s 2>/dev/null || hostname)

# Strips auto-injected context blocks (ide_selection, system-reminder, local
# command wrappers, etc.) so only what you actually typed gets logged.
if ! echo "$input" | jq -c \
  --arg ts "$ts" --arg repo "$repo" --arg branch "$branch" --arg machine "$machine" \
  '
  def clean_prompt:
    gsub("(?i)<(ide_selection|system-reminder|local-command-stdout|local-command-caveat|command-name|command-message|command-args|command-contents|function_results)[^>]*>.*?</\\1>"; ""; "gm")
    | gsub("\n[ \t]*\n[ \t]*\n+"; "\n\n")
    | sub("^\\s+"; "") | sub("\\s+$"; "");
  {timestamp:$ts, repo:$repo, branch:$branch, machine:$machine, session_id:.session_id, prompt:(.prompt | clean_prompt)}
  ' \
  >> "$HOME/.claude/prompt-log.jsonl" 2>>"$errlog"; then
  echo "$ts failed to log prompt (malformed input?)" >> "$errlog"
fi

exit 0
EOF

chmod +x ~/.claude/hooks/log-prompt.sh

SETTINGS=~/.claude/settings.json
[ -f "$SETTINGS" ] || echo '{}' > "$SETTINGS"

if grep -q "log-prompt.sh" "$SETTINGS" 2>/dev/null; then
  echo "Hook already registered in $SETTINGS — skipping."
else
  jq '.hooks.UserPromptSubmit = ((.hooks.UserPromptSubmit // []) + [{"hooks":[{"type":"command","command":"$HOME/.claude/hooks/log-prompt.sh"}]}])' \
    "$SETTINGS" > "$SETTINGS.tmp" && mv "$SETTINGS.tmp" "$SETTINGS"
  echo "Hook registered in $SETTINGS."
fi

echo "✅ Installed. Smoke test:"
echo '{"prompt":"test","session_id":"install-check"}' | ~/.claude/hooks/log-prompt.sh
tail -1 ~/.claude/prompt-log.jsonl
```

## Verify (real session)

Start a new Claude Code session anywhere, submit any prompt, then:
```bash
tail -f ~/.claude/prompt-log.jsonl
```

## Optional: export to human-readable Markdown

Converts the JSONL log into readable Markdown, newest entry first. Safe to re-run —
it tracks a cursor and only processes entries added since the last run, so you can
point a cron/launchd job at it or just run it by hand whenever you want to sync.

```bash
cat > ~/.claude/hooks/prompt-log-to-md.sh << 'EOF'
#!/bin/bash
# Converts ~/.claude/prompt-log.jsonl into a human-readable Markdown file.
# Only processes entries logged since the last run (tracked in a small state file).
# New entries are prepended (newest first), not appended.
#
# Usage: prompt-log-to-md.sh [output_md_path]
# Default output: ~/.claude/prompt-log.md

set -euo pipefail

JSONL="$HOME/.claude/prompt-log.jsonl"
OUT="${1:-$HOME/.claude/prompt-log.md}"
STATE="$HOME/.claude/prompt-log-to-md.state"

[ -f "$JSONL" ] || { echo "No log yet at $JSONL"; exit 0; }

mkdir -p "$(dirname "$OUT")"

# Fixed header block kept at the top of the file: title, generator reminder,
# then a "---" horizontal rule. The blank line before "---" is required so
# Markdown renders a horizontal rule, not a setext (underlined) heading.
HEADER=$(printf '%s\n' \
  '# Claude Code Prompt Log' \
  '' \
  'Generated by CLIO (A member of the rebalanceOS | XYZ | HiQS family)' \
  'https://github.com/Claude-AI-Tools-Ventura-County/clio' \
  '' \
  '---')
HEADER_LINES=6

# Ensure the header exists exactly once. It's written on first creation, and
# only *rebuilt* when the current top-of-file doesn't already match HEADER
# (e.g. a file from an older version) — so repeated syncs never re-insert or
# duplicate it. Entries always begin at the first "## " line, so everything
# above that is treated as the (replaceable) header and the log is preserved.
if [ ! -f "$OUT" ]; then
  printf '%s\n' "$HEADER" > "$OUT"
elif [ "$(head -n "$HEADER_LINES" "$OUT")" != "$HEADER" ]; then
  body=$(sed -n '/^## /,$p' "$OUT")
  { printf '%s\n' "$HEADER"; [ -n "$body" ] && printf '\n%s\n' "$body"; } > "$OUT"
fi

TOTAL_LINES=$(wc -l < "$JSONL" | tr -d ' ')
LAST_LINE=$( [ -f "$STATE" ] && cat "$STATE" || echo 0 )

if [ "$LAST_LINE" -ge "$TOTAL_LINES" ]; then
  echo "Nothing new since last sync ($LAST_LINE/$TOTAL_LINES lines)."
  exit 0
fi

reverse_lines() {
  if command -v tac >/dev/null 2>&1; then tac; else tail -r; fi
}

new_entries=$(mktemp)
tail -n +"$((LAST_LINE + 1))" "$JSONL" | reverse_lines | while IFS= read -r line; do
  echo "$line" | jq -r '
    "## \(.repo | ascii_upcase)\n\(.timestamp)  \n\(.machine)\(if (.branch // "") != "" then " · \(.branch)" else "" end)\n\n> \"\(.prompt | gsub("\n"; "\n> "))\"\n"
  '
done > "$new_entries"

merged=$(mktemp)
{
  head -n "$HEADER_LINES" "$OUT"
  echo
  cat "$new_entries"
  tail -n +"$((HEADER_LINES + 1))" "$OUT"
} > "$merged"
mv "$merged" "$OUT"
rm -f "$new_entries"

echo "$TOTAL_LINES" > "$STATE"
echo "✅ Synced $((TOTAL_LINES - LAST_LINE)) new prompt(s) to $OUT"
EOF

chmod +x ~/.claude/hooks/prompt-log-to-md.sh
```

**Run it** — default location:
```bash
~/.claude/hooks/prompt-log-to-md.sh
```

**Run it** — custom location, e.g. an Obsidian vault:
```bash
~/.claude/hooks/prompt-log-to-md.sh ~/vault/_meta/prompt-log/prompt-log.md
```

The file opens with a fixed header, and each sync prepends entries below it
(newest first):
```markdown
# Claude Code Prompt Log

Generated by CLIO (A member of the rebalanceOS | XYZ | HiQS family)
https://github.com/Claude-AI-Tools-Ventura-County/clio

---

## HYPERCART
2026-07-09T18:42:11Z

Noels-MacBook-Pro · main

> "Help me refactor the wpdbtk delta-sync logic"
```

To auto-sync on a schedule instead of running by hand, add it as a `launchd` job (macOS) or a cron entry pointing at the same command with your chosen output path — the script itself doesn't change either way.

### Auto-sync every 5 minutes (macOS launchd)

Replace `OUT_PATH` with your chosen output file (e.g. an Obsidian note):

```bash
OUT_PATH="$HOME/vault/_meta/prompt-log/prompt-log.md"
PLIST=~/Library/LaunchAgents/com.claude.prompt-log-to-md.plist

cat > "$PLIST" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.prompt-log-to-md</string>
    <key>ProgramArguments</key>
    <array>
        <string>$HOME/.claude/hooks/prompt-log-to-md.sh</string>
        <string>$OUT_PATH</string>
    </array>
    <key>StartInterval</key>
    <integer>300</integer>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>$HOME/.claude/prompt-log-to-md.out.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/.claude/prompt-log-to-md.err.log</string>
</dict>
</plist>
EOF

launchctl unload "$PLIST" 2>/dev/null
launchctl load "$PLIST"
```

Check it's running:
```bash
launchctl list | grep com.claude.prompt-log-to-md
```

Stop and remove it:
```bash
launchctl unload ~/Library/LaunchAgents/com.claude.prompt-log-to-md.plist
rm ~/Library/LaunchAgents/com.claude.prompt-log-to-md.plist
rm -f ~/.claude/prompt-log-to-md.out.log ~/.claude/prompt-log-to-md.err.log
```

## Uninstall

⚠️ This removes only the `log-prompt.sh` entry from `UserPromptSubmit` — it does
**not** touch other `UserPromptSubmit` hooks you may have registered separately.

```bash
tmp=$(mktemp)
jq '.hooks.UserPromptSubmit = ((.hooks.UserPromptSubmit // [])
      | map(.hooks = ((.hooks // []) | map(select(((.command // "") == "$HOME/.claude/hooks/log-prompt.sh") | not))))
      | map(select((.hooks // []) | length > 0)))
    | if (.hooks.UserPromptSubmit // []) == [] then del(.hooks.UserPromptSubmit) else . end' \
  ~/.claude/settings.json > "$tmp" && mv "$tmp" ~/.claude/settings.json

rm ~/.claude/hooks/log-prompt.sh
rm -f ~/.claude/hooks/prompt-log-to-md.sh ~/.claude/prompt-log-to-md.state ~/.claude/prompt-log-errors.log
```

## Notes

- **Scope**: lives in `~/.claude/settings.json` (user-level), so one log file covers every repo — not project-local.
- **`repo`**: git toplevel directory name; falls back to the project directory name if not a git repo.
- **`branch`**: the checked-out branch at prompt time (`git rev-parse --abbrev-ref HEAD`); empty string if not a git repo or in detached HEAD with no symbolic ref. Recorded per-prompt so you can later tell which branch a given ask/change was made on, even after the branch is merged or deleted.
- **`machine`**: macOS "Computer Name" (System Settings → General → Sharing), falling back to `hostname` on other platforms — useful once you're working across more than one machine.
- **Timeout**: `UserPromptSubmit` hooks default to a 30s timeout; a plain append is instant and won't stall a session.
- **Resumed sessions**: `--resume`/`--continue` replay saved context rather than re-running the hook for past turns — only genuinely new prompts get logged.
- **Idempotent**: re-running the install block is safe; it skips re-registering the hook if it's already present, but always rewrites `log-prompt.sh`.
- **MD export is a separate, pull-based step**: it reads the same JSONL and never touches the hook, so the two can be versioned, run, or dropped independently. Delete `prompt-log-to-md.state` if you ever want a full re-export instead of an incremental one.
- **Errors are non-fatal but visible**: the hook always exits 0 so a logging failure never blocks a prompt from being submitted, but any failure (missing `jq`, malformed input) is recorded to `~/.claude/prompt-log-errors.log` instead of vanishing silently. Check that file if entries seem to be missing.
- **Context stripping**: Claude Code's raw prompt field can include auto-injected blocks (`<ide_selection>`, `<system-reminder>`, local-command wrappers) alongside what you actually typed — e.g. having a file selected in your IDE dumps its contents into the prompt. The hook strips known wrapper tags before logging so entries stay a readable record of what you typed, not what the harness injected. jq's regex engine (Oniguruma) uses the `m` flag for dot-matches-newline, not `s` — that's intentional, not a typo.
- **Uninstall matches the exact hook command, not a substring**: it compares each entry to the literal `$HOME/.claude/hooks/log-prompt.sh` string (unexpanded, matching exactly what install writes) rather than using `contains(...)`. A loose substring match would also strip an unrelated command that merely mentions "log-prompt.sh" in its text, and would miss this hook entirely if it were ever bundled into the same matcher block alongside another command at a non-zero array index.
