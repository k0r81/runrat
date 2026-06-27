# 🐀 runrat — dev-tool scavenger & command translator

Finds tools. Translates commands. Learns from failures.

LLM agents constantly generate wrong commands — `apt-get` on Windows, `&&` in PowerShell, invalid flags, Unix tools that don't exist. Runrat sits between the agent and the shell, translating broken commands into what actually works on this machine.

## Install

```bash
# Full bootstrap
npx runrat

# Or via skills.sh
npx skills add k0r81/runrat
```

## What it does

```
Agent: apt-get install postgresql && flutter test --coverage
  ↓
Runrat:
  ⚡ apt-get → scoop (Windows)
  ⚡ && → ; if ($?) { } (PowerShell)
  ⚠ --coverage → --no-pub (valid Flutter flag)
  ↓
Correct: scoop install postgresql; if ($?) { flutter test --no-pub }
```

## Knowledge base

| File | Purpose |
|------|---------|
| `tool-map.json` | Where every tool lives (auto-discovered) |
| `command-recipes.json` | Correct CLI syntax per OS (28 recipes) |
| `translation-rules.json` | LLM mistake patterns → fixes (20 rules) |

## CLI

```bash
runrat check "apt-get install foo"   # translate & validate
runrat setup                         # full bootstrap
runrat recipes                       # list known recipes
runrat rules                         # list active translation rules
```

## Supported translations (Windows)

| LLM writes | runrat corrects to |
|-----------|-------------------|
| `apt-get install X` | `scoop install X` |
| `brew install X` | `scoop install X` |
| `cmd1 && cmd2` | `cmd1; if ($?) { cmd2 }` |
| `export VAR=val` | `$env:VAR = "val"` |
| `pip install X` | `python -m pip install X` |
| `ls -la` | `Get-ChildItem` |
| `rm -rf dir` | `Remove-Item -Recurse -Force dir` |
| `sudo ...` | *(removed)* |
| `which X` | `where.exe X` |

## Platforms

Windows (PowerShell + scoop), macOS (brew), Linux (apt/brew).

## License

MIT
