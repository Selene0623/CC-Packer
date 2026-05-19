# AGENTS.md - CC-Packer

## Project

Fallout 4 Creation Club content merger. PyQt6 GUI that merges individual `cc*.ba2` archives into a few optimized `CCPacked*.ba2` archives with ESL plugins.

## Developer Commands

```bash
pip install -r requirements.txt   # deps (PyQt6, pyinstaller, fake-winreg on non-Windows)
python main.py                     # run GUI
pyinstaller --noconfirm CCPacker.spec  # build exe
```

Build scripts (`build_exe.bat`, `build_release.bat`) are Windows-only. On Linux, run pyinstaller directly.

## Architecture

| File | Role |
|---|---|
| `main.py` | PyQt6 GUI entry point. `MainWindow` (UI) + `MergeWorker`/`RestoreWorker` (QThread) |
| `merger.py` | Core logic. `CCMerger` class: validate, extract, separate, repack, backup, restore |
| `strings_generator.py` | **DEPRECATED** since v1.0.3. Kept for reference only. Do not modify or use. |
| `CCList.txt` | Official CC item database. Loaded at runtime for accurate detection. |
| `CCPacker.spec` | PyInstaller spec. Bundles `bsarch.exe`, `BSARCH_LICENSE.txt`, `CCList.txt`. |

## Key Quirks

- **bsarch.exe is Windows-native**. On Linux it runs via `wine` (see `merger.py:_get_wine_prefix`). A native `bsarch` binary is also searched at `/usr/bin/bsarch` and `/usr/local/bin/bsarch`.
- **`fake-winreg`** provides registry emulation on Linux. Required in `requirements.txt` for non-Windows.
- **No tests, no lint, no typecheck, no CI**. Manual testing only.
- **`plugins.txt`** lives at `%LOCALAPPDATA%/Fallout4/plugins.txt` — Windows-only. Linux FO4 (Proton) uses a different path not yet implemented.
- **Texture split threshold**: 7 GB uncompressed (`merger.py:1181`). Each split gets its own ESL.
- **Output naming**: `CCPacked_*` prefix (v2.0+). Legacy `CCMerged_*` files are cleaned up automatically.
- **Backup dir**: `Data/CC_Backup/<timestamp>/`. Only the most recent backup is kept after restore.
- **Temp dir**: `Data/CC_Temp/` — created during merge, deleted after.

## Merge Flow (merger.py)

1. Validate CC content integrity (plugin + Main.ba2 + Textures.ba2 must all exist)
2. Clean old CCPacked/CCMerged files
3. Timestamped backup of original BA2s to `CC_Backup/`
4. Extract all BA2s via bsarch to `CC_Temp/General` and `CC_Temp/Textures`
5. Move STRINGS files loose to `Data/Strings/`
6. Separate sound files (.xwm/.wav/.fuz/.lip) → uncompressed BA2
7. Repack general content → compressed GNRL BA2
8. Repack textures → compressed DX10 BA2(s), split at 7GB
9. Create minimal ESL plugins (binary TES4 headers, `merger.py:1479`)
10. Add ESLs to `plugins.txt`
11. Delete original CC BA2s, clean temp

## Restore Flow

1. Find most recent backup in `CC_Backup/`
2. Delete all CCPacked/CCMerged files and their STRINGS
3. Remove ESL entries from `plugins.txt`
4. Copy backup files back to Data
5. Delete older backups (keep only latest)

## CC Detection

Uses `CCList.txt` (plugin-first). If unavailable, falls back to glob `cc*.esl/esp/esm` excluding `ccpacked*` prefix.

## Conventions

- PEP 8 style, comprehensive docstrings on all classes/methods
- Type hints throughout
- No external test framework — test manually with real FO4 installation
