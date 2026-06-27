# runrat v2 — Concept Analysis

## 1. THE PROBLEM: How LLMs break commands today

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LLM AGENT (Claude, Copilot, etc.)            │
│                                                                     │
│  User: "zainstaluj postgresa i uruchom testy"                       │
│                                                                     │
│  Agent myśli: "hmm, to pewnie Linux, użyję apt i &&..."             │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  apt-get install │    │  flutter test    │    │  npm run dev  │  │
│  │  postgresql &&   │    │  --coverage      │    │               │  │
│  │  psql -c "..."   │    │  (nie ma flagi!) │    │  (OK)         │  │
│  └──────┬───────────┘    └──────┬───────────┘    └───────┬───────┘  │
│         │                       │                        │          │
└─────────┼───────────────────────┼────────────────────────┼──────────┘
          │                       │                        │
          ▼                       ▼                        ▼
    ┌──────────┐           ┌──────────┐            ┌──────────┐
    │  ❌ FAIL │           │  ❌ FAIL │            │  ✅ OK   │
    │  Windows │           │  zła     │            │          │
    │  nie ma  │           │  flaga   │            │          │
    │  apt-get │           │          │            │          │
    └──────────┘           └──────────┘            └──────────┘
          │                       │
          ▼                       ▼
    ┌──────────────────────────────────────────┐
    │  Agent próbuje jeszcze raz...             │
    │  "apt-get install postgresql-16..."       │
    │  "flutter test --coverage-report..."      │
    │  ❌ FAIL AGAIN                            │
    │  ❌ FAIL AGAIN                            │
    │  User: "daj spokój, sam to zrobię"       │
    └──────────────────────────────────────────┘


    PROBLEM: Agent NIE WIE na jakim systemie działa.
             Agent ZGADUJE składnię.
             Agent NIE UCZY SIĘ na błędach.
```

## 2. THE SOLUTION: runrat as translation layer

```
┌──────────────────────────────────────────────────────────────────────┐
│                         LLM AGENT                                     │
│                                                                      │
│  Agent: "apt-get install postgresql && flutter test --coverage"      │
│                                                                      │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 │  "hey runrat, check this command"
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         🐀 RUNRAT v2                                  │
│                                                                      │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐   │
│  │  tool-map.json  │  │ command-recipes  │  │ translation-rules  │   │
│  │                 │  │     .json        │  │      .json         │   │
│  │  "gdzie jest    │  │                  │  │                    │   │
│  │   psql?"        │  │  "jak uruchomić  │  │  "apt-get → scoop │   │
│  │                 │  │   flutter test?" │  │   && → ; if($?)"  │   │
│  │  adb → C:\...   │  │                  │  │                    │   │
│  │  psql → C:\...  │  │  --fatal-warnings│  │  pip install →    │   │
│  │  flutter → ...  │  │  --no-pub        │  │  python -m pip    │   │
│  └────────┬────────┘  └────────┬─────────┘  └─────────┬──────────┘   │
│           │                    │                       │              │
│           └────────────────────┼───────────────────────┘              │
│                                ▼                                      │
│                    ┌───────────────────────┐                          │
│                    │    TRANSLATE &        │                          │
│                    │    VALIDATE           │                          │
│                    │                       │                          │
│                    │  Input:               │                          │
│                    │  "apt-get install     │                          │
│                    │   postgresql &&       │                          │
│                    │   flutter test        │                          │
│                    │   --coverage"         │                          │
│                    │                       │                          │
│                    │  Output:              │                          │
│                    │  scoop install        │                          │
│                    │  postgresql;          │                          │
│                    │  if ($?) { flutter    │                          │
│                    │  test --no-pub }      │                          │
│                    │                       │                          │
│                    │  Changes:             │                          │
│                    │  ⚡ apt-get→scoop     │                          │
│                    │  ⚡ && → ; if ($?)    │                          │
│                    │  ⚠ --coverage nie    │                          │
│                    │     istnieje dla      │                          │
│                    │     flutter test     │                          │
│                    └───────────┬───────────┘                          │
│                                │                                      │
└────────────────────────────────┼──────────────────────────────────────┘
                                 │
                                 │  poprawiona komenda
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         EXECUTION (bash/zsh/pwsh)                     │
│                                                                      │
│  > scoop install postgresql; if ($?) { flutter test --no-pub }      │
│                                                                      │
│  ✅ postgresql installed                                             │
│  ✅ 42 tests passed                                                  │
│                                                                      │
│  User: 😌                                                           │
└──────────────────────────────────────────────────────────────────────┘
```

## 3. DATA STRUCTURES & RELATIONSHIPS

```
┌─────────────────────────────────────────────────────────────────────┐
│                        KNOWLEDGE BASE                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ tool-map.json                    (v1 — już działa)          │    │
│  │                                                             │    │
│  │  {                                                          │    │
│  │    "adb": { path, source: "android-sdk", category }         │    │
│  │    "flutter": { path, source: "scoop", category }           │    │
│  │    "psql": { path, source: "scoop", category }              │    │
│  │  }                                                          │    │
│  │                                                             │    │
│  │  Rola: "GDZIE jest narzędzie?"                              │    │
│  │  Uczy się: auto-discovery przy setup + runtime              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ command-recipes.json              (v2 — NOWE)               │    │
│  │                                                             │    │
│  │  {                                                          │    │
│  │    "flutter-test": {                                        │    │
│  │      command: "flutter test --no-pub",                      │    │
│  │      platforms: { win32, darwin, linux },                   │    │
│  │      workdir: "app/",                                       │    │
│  │      args: [{ name: "no-pub", flag: "--no-pub" }],          │    │
│  │      category: "flutter"                                    │    │
│  │    }                                                        │    │
│  │  }                                                          │    │
│  │                                                             │    │
│  │  Rola: "JAK poprawnie wywołać narzędzie?"                   │    │
│  │  Źródło: ręcznie curated + auto-generowane z --help         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ translation-rules.json            (v2 — NOWE)               │    │
│  │                                                             │    │
│  │  {                                                          │    │
│  │    patterns: [                                              │    │
│  │      { match: "apt-get install",                            │    │
│  │        platforms: ["win32"],                                │    │
│  │        translate: "scoop install {package}",                │    │
│  │        severity: "error" },                                 │    │
│  │      { match: "&&",                                         │    │
│  │        platforms: ["win32"],                                │    │
│  │        translate: "; if ($?) { {next} }",                   │    │
│  │        severity: "error" }                                  │    │
│  │    ]                                                        │    │
│  │  }                                                          │    │
│  │                                                             │    │
│  │  Rola: "Jaki błąd LLM popełnił i jak go naprawić?"          │    │
│  │  Źródło: ręcznie curated + learning loop z faili            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 4. TRANSLATION PIPELINE (krok po kroku)

```
INPUT: "apt-get install postgresql && flutter test --coverage"


KROK 1: PARSE
─────────────────
┌──────────────────────────────────────────────────┐
│ Rozbij na tokeny:                                │
│                                                  │
│   [apt-get] [install] [postgresql]               │
│   [&&]                                           │
│   [flutter] [test] [--coverage]                  │
│                                                  │
│ Wykryj chaining: &&                              │
│ Wykryj narzędzia: apt-get, flutter               │
│ Wykryj flagi: --coverage                         │
└──────────────────────────────────────────────────┘
                         │
                         ▼
KROK 2: TOOL LOOKUP
─────────────────
┌──────────────────────────────────────────────────┐
│ Sprawdź tool-map.json:                           │
│                                                  │
│   "apt-get" → ❌ not found                       │
│   "flutter" → ✓ C:/.../flutter.bat               │
│   "psql"    → nie użyte jeszcze                  │
│                                                  │
│ Dla apt-get: szukaj w systemie...                │
│   → nie istnieje na Windows                      │
│   → trigger translation rule                    │
└──────────────────────────────────────────────────┘
                         │
                         ▼
KROK 3: TRANSLATE
─────────────────
┌──────────────────────────────────────────────────┐
│ Sprawdź translation-rules.json:                  │
│                                                  │
│   "apt-get install" + platform=win32             │
│   → MATCH: "apt-get install" → "scoop install"   │
│   → severity: error                              │
│                                                  │
│   "&&" + platform=win32                          │
│   → MATCH: "&&" → "; if ($?) { ... }"            │
│   → severity: error                              │
└──────────────────────────────────────────────────┘
                         │
                         ▼
KROK 4: VALIDATE
─────────────────
┌──────────────────────────────────────────────────┐
│ Sprawdź command-recipes.json:                    │
│                                                  │
│   "flutter test" → recipe exists                 │
│   Recipe mówi: "flutter test --no-pub"           │
│                                                  │
│   "--coverage" → ⚠ nie ma w recipe.args          │
│   → ostrzeżenie: flaga może nie istnieć          │
│                                                  │
│   "scoop install" → recipe exists                │
│   → ✓ valid                                      │
└──────────────────────────────────────────────────┘
                         │
                         ▼
KROK 5: OUTPUT
─────────────────
┌──────────────────────────────────────────────────┐
│                                                  │
│   STATUS: ⚡ corrected (3 changes)               │
│                                                  │
│   ORIGINAL:                                      │
│     apt-get install postgresql &&                │
│     flutter test --coverage                      │
│                                                  │
│   CORRECTED:                                     │
│     scoop install postgresql;                    │
│     if ($?) { flutter test --no-pub }            │
│                                                  │
│   CHANGES:                                       │
│     1. ⚡ apt-get → scoop (Windows)              │
│     2. ⚡ && → ; if ($?) (PowerShell)            │
│     3. ⚠ --coverage → --no-pub (valid flag)     │
│                                                  │
│   TOOLS:                                         │
│     scoop   → C:\...\scoop.ps1                   │
│     flutter → C:\...\flutter.bat                 │
│                                                  │
└──────────────────────────────────────────────────┘
```

## 5. LEARNING LOOP

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Agent wykonuje komendę                                            │
│         │                                                           │
│         ▼                                                           │
│   ┌──────────┐     ┌──────────┐                                    │
│   │  ✅ OK   │     │  ❌ FAIL │                                    │
│   └────┬─────┘     └────┬─────┘                                    │
│        │                │                                           │
│        │                ▼                                           │
│        │       ┌─────────────────────────┐                         │
│        │       │ runrat analyze-failure  │                         │
│        │       │                         │                         │
│        │       │ Input:                  │                         │
│        │       │  command: "pip install" │                         │
│        │       │  error: "'pip' is not   │                         │
│        │       │   recognized..."        │                         │
│        │       │                         │                         │
│        │       │ Analysis:               │                         │
│        │       │  1. pip nie istnieje    │                         │
│        │       │     → szukaj python     │                         │
│        │       │  2. python istnieje     │                         │
│        │       │  3. python -m pip       │                         │
│        │       │     działa              │                         │
│        │       │                         │                         │
│        │       │ Action:                 │                         │
│        │       │  → dodaj translation    │                         │
│        │       │    rule:                │                         │
│        │       │    "pip install" →      │                         │
│        │       │    "python -m pip       │                         │
│        │       │     install {pkg}"      │                         │
│        │       │                         │                         │
│        │       │  → zapisz do            │                         │
│        │       │    translation-rules    │                         │
│        │       │    .json                │                         │
│        │       └───────────┬─────────────┘                         │
│        │                   │                                        │
│        │                   ▼                                        │
│        │       ┌─────────────────────────┐                         │
│        │       │ NASTĘPNYM RAZEM:        │                         │
│        │       │                         │                         │
│        │       │ Agent: "pip install x"  │                         │
│        │       │ runrat: ⚡ corrected    │                         │
│        │       │  → python -m pip        │                         │
│        │       │    install x            │                         │
│        │       │                         │                         │
│        │       │ ✅ DZIAŁA              │                         │
│        │       └─────────────────────────┘                         │
│        │                                                           │
│        ▼                                                           │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │  Z czasem translation-rules.json rośnie organicznie       │     │
│   │  z każdym błędem. Agent przestaje popełniać te same       │     │
│   │  błędy.                                                   │     │
│   └──────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 6. WHERE IT SITS IN THE STACK

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   USER: "zainstaluj postgresa i odpal testy"                     │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │              LLM AGENT (Claude, Copilot...)            │     │
│   │                                                        │     │
│   │   "Rozumiem intencję,                      ──►  INTENT │     │
│   │    generuję komendy..."                              │     │
│   └───────────────────────┬────────────────────────────────┘     │
│                           │                                      │
│                           │  surowa komenda                       │
│                           ▼                                      │
│   ┌────────────────────────────────────────────────────────┐     │
│   │                 🐀 RUNRAT                              │     │
│   │                                                        │     │
│   │   "Sprawdzam poprawność,                      ──► FIX  │     │
│   │    tłumaczę na lokalny OS,                             │     │
│   │    waliduję flagi..."                                  │     │
│   └───────────────────────┬────────────────────────────────┘     │
│                           │                                      │
│                           │  poprawiona komenda                  │
│                           ▼                                      │
│   ┌────────────────────────────────────────────────────────┐     │
│   │                   SHELL                                 │     │
│   │                                                        │     │
│   │   "Wykonuję..."                               ──► RUN  │     │
│   └────────────────────────────────────────────────────────┘     │
│                           │                                      │
│                           │  wynik (OK / FAIL)                    │
│                           ▼                                      │
│   ┌────────────────────────────────────────────────────────┐     │
│   │                 🐀 RUNRAT (znowu)                      │     │
│   │                                                        │     │
│   │   FAIL? → "Analizuję błąd,                    ──► LEARN│     │
│   │             zapamiętuję regułę..."                      │     │
│   └────────────────────────────────────────────────────────┘     │
│                                                                  │
│   INTENT ──► FIX ──► RUN ──► LEARN ──► (loop)                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 7. FILE MAP

```
runrat/
│
├── SKILL.md                 ← co skills.sh indeksuje
├── README.md                ← dla ludzi
├── package.json             ← npm: npx runrat
│
├── docs/
│   ├── SPEC.md              ← co i dlaczego
│   ├── AC.md                ← kryteria akceptacji
│   └── CONCEPT.md           ← ten plik (analiza ASCII)
│
├── bin/
│   └── setup                ← CLI (v1: discovery, v2: +recipes +rules)
│
├── knowledge/               ← NOWE: wszystkie pliki wiedzy
│   ├── tool-map.json        ← v1: ścieżki do narzędzi
│   ├── command-recipes.json ← v2: poprawne wywołania per OS
│   └── translation-rules.json ← v2: wzorce błędów LLM
│
└── templates/
    └── dev-runner.md        ← opencode agent (instrukcje dla LLM)
```
