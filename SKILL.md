# 🐀 runrat — dev-tool scavenger

Finds and remembers where every dev tool lives on your machine. Scavenges PATH, package managers, SDKs — builds a smart map that agents can query.

## How it works

1. **Setup** — `npx runrat` scans your system, creates a tool-map.json
2. **Query** — agent reads the map. Tool missing? Scavenges. Found? Hoards it.
3. **Repeat** — safe to re-run, always learns more

## Resolve any tool

### 1. Check the map

Read `.opencode/agents/tool-map.json`. Return `path` if found.

### 2. Search the system

Stop on first hit:

**Linux / macOS:**
```bash
which <tool> 2>/dev/null || command -v <tool>
ls /opt/homebrew/bin/<tool> /usr/local/bin/<tool> 2>/dev/null
ls /usr/bin/<tool> /usr/local/bin/<tool> ~/.local/bin/<tool> 2>/dev/null
find ~/Android/Sdk ~/Library/Android/sdk -name "<tool>" -type f 2>/dev/null | head -1
```

**Windows:**
```powershell
where.exe <tool> 2>$null | Select-Object -First 1
Get-ChildItem "$env:USERPROFILE\scoop\shims" -Filter "<tool>*" -ErrorAction SilentlyContinue | % FullName
Get-ChildItem "$env:USERPROFILE\scoop\apps" -Filter "<tool>.exe" -Recurse -Depth 3 -ErrorAction SilentlyContinue | % FullName
Get-ChildItem "$env:LOCALAPPDATA\Android\Sdk","$env:USERPROFILE\Android\Sdk" -Filter "<tool>.exe" -Recurse -Depth 4 -ErrorAction SilentlyContinue | % FullName
```

### 3. Save learned tool

Add to `tool-map.json`:

```json
"toolname": {
  "path": "/absolute/path",
  "source": "scoop|brew|android-sdk|system|path",
  "category": "..."
}
```

### 4. Not found

"Tool `<tool>` not found. Searched: PATH, package manager dirs, Android SDK."

## Tool categories

| Category | Tools |
|----------|-------|
| `android` | adb, emulator, sdkmanager |
| `flutter` | flutter, dart |
| `vcs` | git, gh |
| `node` | node, npm, npx, yarn, pnpm |
| `python` | python, python3, pip, pip3 |
| `java` | java, javac, gradle, mvn |
| `database` | supabase, psql, sqlite3 |
| `container` | docker, podman |
| `editor` | code, nvim, vim |
| `package-manager` | scoop, choco, brew, apt, winget |
| `build` | make, cmake |
| `other` | everything else |
