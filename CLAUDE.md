# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FlatCAM Evo is a Python CAM (Computer-Aided Manufacturing) application for preparing CNC jobs to make PCBs. It imports Gerber/Excellon/DXF/SVG files and generates G-Code for isolation routing, drilling, milling, and other PCB manufacturing tasks.

- **Language:** Python 3.6+ (current environment uses 3.11)
- **GUI:** PyQt6
- **Graphics:** VisPy (3D engine), Matplotlib (legacy)
- **Geometry:** Shapely, numpy, scipy
- **Entry point:** `flatcam.py`

## Running the Application

```bash
python flatcam.py                          # Normal launch
python flatcam.py --headless=1             # Headless mode (no GUI)
python flatcam.py --shellfile=<script>     # Execute TCL script on startup
python flatcam.py --shellvar=<values>      # Pass variables to script
```

User data directory: `~/.FlatCAM/` (Unix) or `%APPDATA%\FlatCAM` (Windows).
Log file: `~/.FlatCAM/log.txt`.

## Building Documentation

```bash
cd doc && make html    # Build Sphinx docs
```

## Architecture

### Core Modules (root level)

- **appMain.py** (~8k lines) — Main `App` class (QObject). Central orchestrator: manages object lifecycle, signal/slot wiring, CLI args, version management. This is the largest single module.
- **camlib.py** (~8.5k lines) — Core geometry and manufacturing library. Contains `ApertureMacro`, `Geometry`, `CNCjob`, and `AppRTree` classes. All geometric operations and G-code generation live here.
- **defaults.py** — `AppDefaults` class with `factory_defaults` dict. All preference keys and their defaults.
- **appDatabase.py** — Tool database management (diameters, feeds, speeds).
- **appTool.py** — Base class for all tool plugins.
- **appWorker.py / appPool.py** — Threading (QThread) and multiprocessing pool.
- **appTranslation.py** — i18n wrapper around gettext. Strings use `_()` builtin.

### Subsystem Directories

| Directory | Purpose |
|---|---|
| `appGUI/` | Main window (`MainGUI.py`), custom widgets (`GUIElements.py`), canvas (`PlotCanvas3d.py`), themes, preferences panels |
| `appObjects/` | Data model classes: `GerberObject`, `ExcellonObject`, `GeometryObject`, `CNCJobObject`, `DocumentObject`, `ScriptObject`. All inherit from `AppObjectTemplate`. `ObjectCollection` manages them. |
| `appEditors/` | Interactive editors for Gerber, Excellon, Geometry, G-Code, and text. Each editor has sub-plugins in `*_plugins/` dirs. |
| `appHandlers/` | `appIO.py` (file import/export), `appEdit.py` (edit operations on objects) |
| `appParsers/` | File format parsers: `ParseGerber`, `ParseExcellon`, `ParseDXF`, `ParseSVG`, `ParsePDF`, `ParseHPGL2`, `ParseFont` |
| `appPlugins/` | ~39 manufacturing tool plugins (isolation, NCC, paint, milling, drilling, cutout, panelize, levelling, film, solder paste, fiducials, rules check, etc.) |
| `tclCommands/` | ~79 TCL command modules for the scripting interface. Base class: `TclCommand.py`. |
| `preprocessors/` | ~20 G-code post-processors (GRBL variants, Marlin, LinuxCNC, Roland, etc.) |
| `locale/` | Translations for 10 languages |
| `appCommon/` | Shared utilities (`Common.py`), bilinear interpolation |

### Key Patterns

- **Object model:** All project items (Gerber, Excellon, Geometry, CNCJob) are subclasses of `AppObjectTemplate` in `appObjects/`. They're managed by `ObjectCollection` which backs the GUI project tree.
- **Plugin/tool architecture:** Tools in `appPlugins/` inherit from `AppTool`. Each tool typically has a UI panel, processing logic, and an associated TCL command.
- **Signal/slot communication:** PyQt signals drive the event system. `LoudDict` (in `appCommon/Common.py`) and `FCSignal` provide custom observable patterns.
- **Threading model:** Long operations run on `QThread` via `appWorker.py`. CPU-bound parallel work uses `multiprocessing.Pool` via `appPool.py`. Never access GUI from worker threads.
- **File import flow:** `appIO.py` detects format → appropriate parser loads geometry → object created and added to `ObjectCollection` → rendered on VisPy canvas.
- **Preferences:** Stored via `QSettings("Open Source", "FlatCAM_EVO")`. Defaults defined in `defaults.py`. Tools read from the shared options dict.
- **Localization:** All user-facing strings wrapped in `_()`. Translation catalogs in `locale/`. New strings need `gettext` extraction.

### No Automated Test Suite

The project does not have an integrated test framework (no pytest/unittest). Testing is manual. If adding tests, place them alongside the relevant module or in a new `tests/` directory.
