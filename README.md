# 🐀 runrat

[![skills.sh](https://skills.sh/b/k0r81/runrat)](https://skills.sh/k0r81/runrat)
[![npm](https://img.shields.io/npm/v/runrat)](https://www.npmjs.com/package/runrat)

> The dev-tool scavenger rat. Sniffs, hoards, remembers.

Runrat scampers through your system — PATH, scoop shims, Android SDK, brew cellars — finds every dev tool and hoards their paths. Installs as an **opencode subagent** that learns on the fly.

## Install

```bash
# Quick (skills.sh)
npx skills add k0r81/runrat

# Full setup with auto-discovery
npx runrat
```

## What it finds

adb · flutter · dart · git · node · npm · npx · python · pip · java · javac · gradle · sqlite3 · docker · gh · code · supabase · make · cmake · and anything else you throw at it

## How it works

```
npx runrat
  → scans system
  → writes .opencode/agents/dev-runner.md
  → writes .opencode/agents/tool-map.json
  → restart opencode, agent is live
```

Agent reads `tool-map.json` first. Tool missing? Scavenges. Finds it? Hoards it in the map forever.

## Platforms

| OS | Where it looks |
|----|---------------|
| Windows | `where.exe`, PowerShell, scoop shims/apps, Android SDK |
| macOS | `which`, brew (`/opt/homebrew`), Android SDK |
| Linux | `which`, `/usr/bin`, `~/.local/bin`, Android SDK |

## License

MIT
