# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

An OpenClaw skill that scans JSONL session logs (`~/.openclaw/agents/main/sessions/`) for leaked credentials. It distinguishes real leaks (credentials sent to an AI provider) from "config echoes" (the config file itself appearing in a session log).

## Running

```bash
node scripts/leak-check.js                    # Discord-formatted output
node scripts/leak-check.js --format json       # JSON output
node scripts/leak-check.js --config /path.json # Custom config
```

No build step, no dependencies beyond Node.js.

## Architecture

Single-file Node.js script (`scripts/leak-check.js`) with this flow:

1. **Load credentials** from `leak-check.json` (checked in skill dir first, then `~/clawd/`, overridable with `--config`) — array of `{name, search}` with wildcard pattern support
2. **Walk session directory** recursively for `.jsonl` files (including `.jsonl.deleted.*`, `.jsonl.reset.*`)
3. **Per file**: track the current real provider/model from `model_change` and cost-bearing `message` events, then match each credential pattern against raw line text
4. **Config echo detection**: if a file contains the JSON config definition of a credential (`"search": "VALUE"`), matches in that file are flagged as config echoes rather than real leaks
5. **Dedup** by credential+session (keep earliest timestamp), then format output

Key design decisions:
- Matching runs against raw line text (not parsed JSON) for speed; JSON is only parsed for metadata extraction
- Provider `"openclaw"` is filtered out via `isRealProvider()` — only external providers count as leaks
- Wildcard `*` in search patterns matches `[^\s"]{0,200}` (non-whitespace/non-quote, capped at 200 chars to avoid base64 blob false positives)

## Config Format

`leak-check.json` contains **partial** credential fragments, not full secrets. Wildcard patterns: `abc*xyz` (starts+ends), `*xyz` (ends-with), `abc*` (starts-with), `abc` (contains).

The recommended persistent location is `~/clawd/leak-check.json` since clawhub clears the skill directory on updates.
