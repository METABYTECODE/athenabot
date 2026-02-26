# AGENTS.md

## Cursor Cloud specific instructions

This repository (`METABYTECODE/athenabot`) is a **content-only documentation repository**. It contains no source code, no build system, and no runnable application.

### Repository contents

| File | Description |
|------|-------------|
| `updates.json` | Release/update manifest pointing to a pre-compiled Windows executable (`BotGUI.exe`) hosted on GitHub Releases, with license keys and expiration timestamps |
| `info.md` | Russian-language documentation describing game damage calculation formulas (references Lua vscripts not present in this repo) |
| `armor.md` | Russian-language documentation describing armor and corrosion mechanics |

### Development notes

- There are **no dependencies** to install (no `package.json`, `requirements.txt`, etc.).
- There are **no tests**, **no linting**, and **no build steps**.
- There is **no application to run** — the actual bot binary (`BotGUI.exe`) is distributed via GitHub Releases and its source code is not in this repository.
- The referenced Lua game scripts (`mechanics/damage_system.lua`, etc.) are not present in this repo; `info.md` and `armor.md` document their behavior externally.
- Content is written in Russian.
- Validation of this repo is limited to checking that the JSON is well-formed and the Markdown files are present.
