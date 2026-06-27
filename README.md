# 🐀 runrat

[![skills.sh](https://skills.sh/b/k0r81/runrat)](https://skills.sh/k0r81/runrat)
[![npm](https://img.shields.io/npm/v/runrat)](https://www.npmjs.com/package/runrat)

> The dev-tool scavenger rat. Sniffs, hoards, remembers. Fixes commands before they break.

## The problem

LLM coding agents (Claude, Copilot, Cursor) are **OS-blind**. They generate commands that work on the machine they were trained on — usually Linux with bash. On your actual machine:

```
Agent:  apt-get install postgresql && pip install django && export SECRET=foo
User:   ❌ 'apt-get' is not recognized
        ❌ 'export' is not recognized
        ❌ pip uses wrong Python
```

The agent doesn't know you're on Windows with PowerShell and scoop. It doesn't know `flutter test` needs `--no-pub` not `--coverage`. It doesn't know `sudo` doesn't exist here.

Every failed command costs you:
- Time (agent retries with another wrong command)
- Trust (you stop believing the agent can do anything)
- Sanity

## The solution

Runrat sits between the agent and the shell. Before any command runs, runrat checks and translates it:

```bash
runrat check "apt-get install postgresql && flutter test --coverage"
```

```
⚡ apt-get → scoop install (Windows has no apt)
⚡ && → ; if ($?) { } (PowerShell has no &&)
⚠ --coverage → --no-pub (valid Flutter flag)

corrected: scoop install postgresql; if ($?) { flutter test --no-pub }
```

## How it works

Three knowledge files that grow over time:

| File | What it knows | Size |
|------|--------------|------|
| `translation-rules.json` | LLM mistake patterns → correct equivalents | 20 rules |
| `tool-map.json` | Where every tool lives (auto-discovered) | per machine |
| `command-recipes.json` | Correct CLI syntax and flags per OS | 28 recipes |

## Install

```bash
# Full bootstrap
npx runrat

# Or via skills.sh
npx skills add k0r81/runrat
```

Restart opencode. The dev-runner subagent is now active.

## CLI

```bash
runrat check "apt-get install foo"   # Translate & validate any command
runrat setup                         # Bootstrap on this machine
runrat recipes                       # List all known command recipes
runrat rules                         # List active translation rules
```

## Examples of what it fixes

| LLM writes | OS | Corrected to |
|-----------|-----|-------------|
| `apt-get install X` | Windows | `scoop install X` |
| `apt-get install X` | macOS | `brew install X` |
| `brew install X` | Windows | `scoop install X` |
| `scoop install X` | macOS/Linux | `brew install X` |
| `cmd1 && cmd2` | Windows | `cmd1; if ($?) { cmd2 }` |
| `export VAR=val` | Windows | `$env:VAR = "val"` |
| `pip install X` | All | `python -m pip install X` |
| `sudo npm -g X` | macOS/Linux | `npm install -g X` |
| `ls -la` | Windows | `Get-ChildItem` |
| `rm -rf dir` | Windows | `Remove-Item -Recurse -Force dir` |
| `python3` | Windows | `python` |
| `which X` | Windows | `where.exe X` |
| `touch file` | Windows | `New-Item -ItemType File file` |
| `cat file` | Windows | `Get-Content file` |
| `grep foo` | Windows | `Select-String foo` |
| `flutter test` | All | `flutter test --no-pub` |
| `dart format .` | All | `dart format --set-exit-if-changed lib/ test/` |

## Platforms

Windows (PowerShell + scoop), macOS (brew), Linux (apt/brew).

## License

MIT
