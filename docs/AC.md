# AC: runrat v2 MVP

## Core: command translation

- [ ] `runrat check "apt-get install postgresql"` na Windows → zwraca `scoop install postgresql`
- [ ] `runrat check "flutter analyze"` → zwraca `flutter analyze --fatal-warnings` (bo recipe)
- [ ] `runrat check "dart format ."` → zwraca `dart format --set-exit-if-changed lib/ test/` (bo recipe)
- [ ] `runrat check "npm install -g && npm start"` na Windows → tłumaczy `&&` na `; if ($?) { ... }`
- [ ] `runrat check "sudo pip install django"` → ostrzega: `sudo` niepotrzebne, użyj `python -m pip install django`
- [ ] `runrat check "ls -la"` na Windows → tłumaczy na `Get-ChildItem` lub `dir`

## Recipes

- [ ] `command-recipes.json` zawiera co najmniej 10 przepisów dla Flutter/Dart
- [ ] `command-recipes.json` zawiera przepisy dla git (add, commit, push, status, log)
- [ ] `command-recipes.json` zawiera przepisy dla npm/node
- [ ] `command-recipes.json` zawiera przepisy dla adb
- [ ] Każdy recipe ma `platforms` (win32, darwin, linux) z poprawną składnią
- [ ] `runrat setup` generuje recipes dopasowane do lokalnego OS

## Translation rules

- [ ] `translation-rules.json` zawiera co najmniej 5 zasad
- [ ] Zasada: apt-get → scoop (Windows) / brew (macOS)
- [ ] Zasada: && → ; if ($?) { } (Windows PowerShell)
- [ ] Zasada: pip install → python -m pip install (bezpieczniejsze)
- [ ] Zasada: sudo npm -g → npm -g (ostrzeżenie)
- [ ] Każda zasada ma `severity` (error/warning/info)

## Agent behavior

- [ ] Agent przyjmuje komendę jako input, zwraca: oryginalna komenda + poprawiona + wyjaśnienie zmian
- [ ] Agent sprawdza tool-map.json dla każdego narzędzia w komendzie
- [ ] Jeśli narzędzia nie ma w mapie → szuka w systemie → zapisuje
- [ ] Agent raportuje: `✓ valid` / `⚡ corrected (N changes)` / `✗ unknown tool`
- [ ] Agent NIGDY nie wykonuje komendy — tylko tłumaczy i zwraca

## CLI

- [ ] `runrat check "<command>"` — sprawdza i tłumaczy pojedynczą komendę
- [ ] `runrat setup` — bootstrap (tool discovery + recipes + translation rules)
- [ ] `runrat recipes` — lista znanych przepisów
- [ ] `runrat rules` — lista aktywnych reguł tłumaczenia

## Cross-platform

- [ ] Wszystkie komendy w recipes mają warianty per platform
- [ ] Translation rules są per-platform (match może być globalny lub specyficzny)
- [ ] Setup działa na Windows, macOS, Linux
