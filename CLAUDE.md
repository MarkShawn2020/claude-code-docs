# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A community mirror of Claude Code documentation from https://docs.anthropic.com/en/docs/claude-code/, providing local access via the `/docs` slash command. Docs are auto-synced every 3 hours via GitHub Actions.

## Architecture

```
├── docs/                    # Mirrored markdown documentation files
│   └── docs_manifest.json   # Tracks file hashes, URLs, fetch metadata
├── scripts/
│   ├── fetch_claude_docs.py           # Python fetcher (runs in GitHub Actions)
│   └── claude-docs-helper.sh.template # Template for helper script
├── install.sh               # Installs to ~/.claude-code-docs
├── uninstall.sh             # Removes installation
└── .github/workflows/
    └── update-docs.yml      # Scheduled docs sync (every 3 hours)
```

**Installation Flow**: `install.sh` → clones to `~/.claude-code-docs` → creates `/docs` command in `~/.claude/commands/docs.md` → adds PreToolUse hook to `~/.claude/settings.json`

**Docs Fetch Flow**: GitHub Actions → `fetch_claude_docs.py` → discovers pages from sitemap → fetches markdown → saves to `docs/` → updates manifest

## Key Scripts

- **install.sh**: Handles fresh install, migration from old locations, git updates with conflict resolution. Version: 0.3.3
- **scripts/fetch_claude_docs.py**: Discovers docs via sitemap XML parsing, fetches markdown, validates content, manages manifest with content hashing
- **claude-docs-helper.sh** (generated): Runtime script handling `/docs` commands, auto-updates, freshness checks

## Testing Changes

```bash
# Test install script locally (installs to ~/.claude-code-docs)
./install.sh

# Test docs fetcher
cd scripts && python fetch_claude_docs.py

# Test helper script
~/.claude-code-docs/claude-docs-helper.sh hooks
~/.claude-code-docs/claude-docs-helper.sh -t  # freshness check
```

## Important Behaviors

- **Shell compatibility**: Scripts must work in both bash and zsh (macOS default)
- **Error handling**: All scripts use `set -euo pipefail`
- **Input sanitization**: Helper script sanitizes user input to prevent command injection
- **Manifest preservation**: `docs_manifest.json` is git-tracked; don't modify during install
- **Old installation cleanup**: Installer finds and migrates from any previous install location
