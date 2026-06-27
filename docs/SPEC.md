# SPEC: runrat v2 — Command Translator for LLM Agents

## Problem

LLM agents (Claude, Copilot, Cursor, etc.) frequently generate **broken commands**:

- `apt-get install` na Windows zamiast `scoop install`
- `&&` chainowanie na PowerShell (nie działa)
- `python` zamiast `python3`, `pip` zamiast `pip3`
- nieistniejące flagi CLI (`flutter analyze --watch` — nie ma takiej)
- Unixowe ścieżki na Windows (`/usr/bin/...`)
- zgadywanie nazw narzędzi zamiast sprawdzenia co faktycznie jest zainstalowane

Efekt: user dostaje błąd, agent próbuje jeszcze raz z inną złą komendą, frustracja rośnie.

## Vision

**runrat v2** staje się warstwą tłumaczącą komendy LLM na poprawne komendy dla tej konkretnej maszyny.

```
Agent generuje:  apt-get install postgresql
                    ↓
        runrat sprawdza OS (win32)
                    ↓
runrat zwraca:    scoop install postgresql   ← poprawna komenda dla tego systemu
```

runrat zna:
- jakie narzędzia są zainstalowane (tool-map.json)
- jak poprawnie ich używać na każdym OS (command-recipes.json)
- typowe błędy LLM i jak je naprawiać (translation-rules.json)

## Architecture v2

```
runrat/
├── SKILL.md                  ← agent instructions
├── tool-map.json             ← { tool: { path, source, category } }  [v1]
├── command-recipes.json      ← { command_id: { platforms, workdir } } [v2 NEW]
├── translation-rules.json    ← { patterns: [{ match, translate }] }  [v2 NEW]
├── bin/setup                 ← CLI bootstrap
└── templates/
    └── dev-runner.md         ← opencode agent template
```

### tool-map.json (v1 — już istnieje)

```json
{
  "adb": {
    "path": "C:/Users/korba/Android/Sdk/platform-tools/adb.exe",
    "source": "android-sdk",
    "category": "android"
  }
}
```

### command-recipes.json (v2 — NOWE)

Kanoniczne przepisy na poprawne wywołanie każdego narzędzia, per platform.

```json
{
  "flutter-analyze": {
    "description": "Run Flutter static analysis",
    "platforms": {
      "win32":  { "command": "flutter analyze --fatal-warnings", "shell": "powershell" },
      "darwin": { "command": "flutter analyze --fatal-warnings", "shell": "bash" },
      "linux":  { "command": "flutter analyze --fatal-warnings", "shell": "bash" }
    },
    "workdir": "app/",
    "category": "flutter",
    "args": [
      { "name": "fatal-warnings", "flag": "--fatal-warnings", "required": false, "description": "Treat warnings as errors" }
    ]
  },
  "flutter-test": {
    "description": "Run Flutter tests",
    "platforms": {
      "win32":  { "command": "flutter test --no-pub", "shell": "powershell" },
      "darwin": { "command": "flutter test --no-pub", "shell": "bash" },
      "linux":  { "command": "flutter test --no-pub", "shell": "bash" }
    },
    "workdir": "app/",
    "category": "flutter"
  },
  "dart-format": {
    "description": "Format Dart source files",
    "platforms": {
      "win32":  { "command": "dart format --set-exit-if-changed lib/ test/", "shell": "powershell" },
      "darwin": { "command": "dart format --set-exit-if-changed lib/ test/", "shell": "bash" },
      "linux":  { "command": "dart format --set-exit-if-changed lib/ test/", "shell": "bash" }
    },
    "workdir": "app/",
    "category": "flutter"
  }
}
```

### translation-rules.json (v2 — NOWE)

Wzorce błędów LLM → poprawne odpowiedniki.

```json
{
  "patterns": [
    {
      "id": "apt-on-windows",
      "match": "apt-get install",
      "platforms": ["win32"],
      "translate": "scoop install {package}",
      "category": "package-manager",
      "severity": "error"
    },
    {
      "id": "apt-get-on-mac",
      "match": "apt-get install",
      "platforms": ["darwin"],
      "translate": "brew install {package}",
      "category": "package-manager",
      "severity": "error"
    },
    {
      "id": "double-ampersand-win",
      "match": "&&",
      "platforms": ["win32"],
      "translate": "; if ($?) { {next} }",
      "category": "shell-syntax",
      "severity": "error"
    },
    {
      "id": "pip-without-python",
      "match": "^pip install",
      "platforms": ["win32", "darwin", "linux"],
      "translate": "{python} -m pip install {package}",
      "category": "python",
      "severity": "warning"
    },
    {
      "id": "npm-global-no-sudo",
      "match": "sudo npm install -g",
      "platforms": ["darwin", "linux"],
      "translate": "npm install -g {package}",
      "category": "node",
      "severity": "warning"
    }
  ]
}
```

## Agent flow v2

```
Agent calls runrat with: "run flutter analyze"

1. Look up "flutter analyze" in command-recipes.json
   → found: { win32: "flutter analyze --fatal-warnings", workdir: "app/" }
2. Check tool-map.json for flutter path
   → found: C:/Users/korba/scoop/apps/flutter/current/bin/flutter
3. Check translation-rules.json for any patterns that match
   → no matches (command is already correct)
4. Return:
   command: flutter analyze --fatal-warnings
   tool:    flutter (C:/Users/korba/scoop/apps/flutter/current/bin/flutter)
   workdir: app/
   status:  ✓ valid
```

```
Agent calls runrat with: "apt-get install postgresql && psql -V"

1. Check translation-rules.json
   → MATCH: "apt-get install" on win32 → "scoop install {package}"
   → MATCH: "&&" on win32 → "; if ($?) { psql -V }"
2. Check tool-map.json for psql
   → not found, search system... found at C:/Program Files/PostgreSQL/16/bin/psql.exe
   → saved to tool-map.json
3. Return:
   original:  apt-get install postgresql && psql -V
   corrected: scoop install postgresql; if ($?) { psql -V }
   tool:      psql (C:/Program Files/PostgreSQL/16/bin/psql.exe)
   changes:   2 translations applied
   status:    ⚡ corrected
```

## Learning loop

1. Agent runs command, it fails
2. Agent calls runrat with the failed command + error output
3. runrat analyzes: wrong flag? missing tool? syntax error?
4. runrat finds or creates a translation rule
5. Rule saved to translation-rules.json for next time
6. Returns corrected command

## Scope

### v2 MVP (co robimy teraz)

- [x] tool discovery (tool-map.json) — z v1
- [ ] command-recipe knowledge base z poprawną składnią per OS
- [ ] translation rules dla najczęstszych błędów LLM
- [ ] agent potrafi tłumaczyć komendy przed wykonaniem
- [ ] komenda `runrat check "apt-get install foo"` → zwraca poprawioną wersję
- [ ] `runrat setup` — bootstrapuje recipes i translation rules dla danego systemu

### v2.1 (przyszłość)

- [ ] learning from failures (feedback loop)
- [ ] auto-detection of new tools and their CLI flags
- [ ] `runrat explain <command>` — wyjaśnia co komenda robi
- [ ] integracja z CI: `runrat validate` sprawdza wszystkie komendy przed wykonaniem
