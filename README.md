# NK Advanced Collect  **A multi-threaded asset collection and archiving tool for Foundry Nuke**




<div align="center">

# NK Advanced Collect

**A multi-threaded asset collection and archiving tool for Foundry Nuke**

![Platform](https://img.shields.io/badge/platform-Nuke%2012%E2%80%9317-76B900)
![Qt](https://img.shields.io/badge/Qt-PySide2%20%7C%20PySide6-41cd52)
![Python](https://img.shields.io/badge/python-3.x-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Scans a script for every file dependency, copies it into a self-contained archive,
relinks the script with relative paths, and produces an auditable manifest —
so the project can move between artists, sites, or studios without a single
broken path.

</div>

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
  - [From the panel](#from-the-panel)
  - [From the Script Editor](#from-the-script-editor)
  - [Headless / farm mode](#headless--farm-mode)
- [Configuration](#configuration)
- [Output](#output)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Credits](#credits)
- [License](#license)

---

## Overview

NK Advanced Collect walks a Nuke script's node graph and identifies every file
dependency — footage, geometry, cameras, audio, precomps, LUTs, OCIO configs, and
fonts — then copies each one into a `footage/` folder next to the saved script and
relinks the collecting nodes with a relative `[file dirname [value root.name]]`
expression. The result is a self-contained, portable project folder, with a full
`manifest.json` and `archive_report.txt` documenting exactly what was collected.

Originally based on **Collect Files** by Nitesh Pancholi. Rewritten and extended
(v3.1) for Nuke 16/17 by **Nitin Kashyap**, adding a dockable Qt panel, batch
archiving, dry-run pre-flight reports, and Nuke 17 new-3D-system (USD) support.

---

## Features

| Category | Details |
|---|---|
| **Dockable Qt panel** | `Windows > Custom > NK Advanced Collect`. PySide6 on Nuke 15+, PySide2 on older builds — auto-detected. |
| **Batch mode** | Queue multiple `.nk` scripts and collect them in a single run, each into its own subfolder. |
| **Dry-run report** | Pre-flight summary of file count, total size, and missing files *before* anything is copied. |
| **Include / exclude filters** | Toggle audio, LUTs/OCIO config, fonts, Write/render outputs, and gizmo/group recursion independently. |
| **Dependency coverage** | Read, ReadGeo, Camera, Axis, AudioRead, Precomp, Vectorfield, LUTs, OCIO config, fonts on Text/Text2, and Nuke 17 `GeoReference` (USD/Alembic/FBX). |
| **Frame-range aware** | Respects a Read's local/cropped frame range, or a custom global override, instead of always copying first–last. |
| **Multi-threaded copy** | Live byte-level progress, plus pause / resume / cancel. |
| **Auditable output** | `manifest.json` (machine-readable) and `archive_report.txt` (human-readable), each with a unique job ID. |
| **Search Missing Files** | Scans all Read nodes for broken paths, searches the local machine, and offers to relink. |
| **CLI / farm mode** | `collectFiles_cli()` runs headlessly (e.g. via `nuke -t`) — no panels, safe for render-farm automation. |
| **Nuke 17 support** | `GeoExport` is correctly treated as a render output (not a source); `GeoReference` is collected/relinked via its `file_path` knob. |

**Supported video formats:** `mov` · `avi` · `mpeg` · `mpg` · `mp4` · `r3d`

---

## Installation

1. **Copy the plugin folder**

   Copy the whole `NK Advanced Collect` folder into your Nuke plugins path:

   | OS | Path |
   |---|---|
   | Windows | `C:\Users\<you>\.nuke\NK Advanced Collect` |
   | macOS / Linux | `~/.nuke/NK Advanced Collect` |

   Any folder already on your `NUKE_PATH` also works.

2. **Register the path** (only if installed outside your default `~/.nuke`)

   Add to your `init.py`:

   ```python
   nuke.pluginAddPath(r"NK Advanced Collect")
   ```

3. **Menu registration**

   The included `menu.py` registers everything automatically on startup:
   - **`NK Advanced Collect`** menu (top menu bar + Nodes menu) — panel, Search
     Missing Files, About.
   - The panel as a dockable pane under **`Windows > Custom > NK Advanced Collect`**.

   If you already maintain your own `menu.py`, copy the relevant blocks from the
   included one into yours rather than overwriting it.

4. **Restart Nuke.**

> **Note:** Settings and the recent-scripts list live under
> `~/.nuke/NK Advanced Collect/database/` and are created automatically the first
> time you open the panel or change a setting. The `database/` folder shipped in
> this package is a placeholder and can be safely deleted.

---

## Usage

### From the panel

Open **`Windows > Custom > NK Advanced Collect`** (or the top menu entry). Add one
or more scripts to the queue, choose an output directory, adjust include/exclude
and archive settings via the gear icon, then click **Collect**. Progress, per-file
status, pause/resume, and cancel are all available directly in the panel.

### From the Script Editor

```python
import collectFiles

collectFiles.collectFiles()          # interactive: options panel + dry-run report
collectFiles.search_missing_files()  # relink broken Read paths
```

### Headless / farm mode

```python
import collectFiles

collectFiles.collectFiles_cli("/path/to/output", options={
    "audio":   True,
    "luts":    True,
    "fonts":   True,
    "renders": False,
    "gizmos":  True,
})
```

Runs with no panels; safe to call from a farm job or pipeline hook.

---

## Configuration

All options are available both in the panel's settings dialog (gear icon) and as
keys in the `options` dict passed to `collectFiles_cli()`:

| Option | Default | Description |
|---|---|---|
| `audio` | `True` | Collect `AudioRead` files. |
| `luts` | `True` | Collect LUT files (`.cube`, etc.) and the OCIO config. |
| `fonts` | `True` | Collect fonts referenced by `Text` / `Text2` nodes. |
| `renders` | `False` | Also collect `Write` node output paths. |
| `gizmos` | `True` | Recurse into Group/Gizmo nodes to find nested Reads. |
| `recreate_source_paths` | `False` | Rebuild each file's original folder structure under `footage/` instead of a flat, category-based dump. |
| `max_copy_workers` | `8` | Thread pool size for the copy phase. |
| `comment` | `""` | Free-text note stored in `archive_report.txt`. |

---

## Output

A collect run produces, next to the archived script:

```
<output_dir>/
├── footage/              # every collected dependency
├── manifest.json         # machine-readable record: src, dst, size, status
├── archive_report.txt    # human-readable summary with job ID + settings
└── <script_name>.nk      # the relinked, saved script
```

---

## Requirements

- Nuke 17.0v3 or newer recommended. Core collection logic also works on Nuke 12+;
  the Nuke 17 USD-node handling (`GeoReference`, `GeoExport`) only applies if
  you're on 17.x.
- PySide2 or PySide6 — ships with Nuke, no separate install needed.
- `QtMultimedia` *(optional)* — enables the "collect finished" sound cue. If
  unavailable in your Nuke build, sound playback is silently skipped.

---

## Project Structure

```
NK Advanced Collect/
├── collectFiles.py       # Core scan / manifest / copy / relink logic (no Qt dependency beyond nuke.Panel)
├── collectFilesUI.py     # Dockable Qt panel (front end) — calls into collectFiles.py for all real work
├── menu.py                # Registers menu items + dockable panel on Nuke startup
├── icons/                 # UI icon assets
├── Logo/                  # Branding assets
├── sounds/                 # "Collect finished" audio cue
└── database/               # Auto-populated at runtime (settings, recent scripts) under ~/.nuke
```

---

## Credits

- Original **Collect Files** script by **Nitesh Pancholi**.
- Rewritten and extended (v3.0/3.1 — Qt panel, batch mode, Nuke 17 support) by
  **Nitin Kashyap**.
---

<div align="center">

*If this tool saves you time on a project, consider crediting the author when
sharing it with your team or studio.*

</div>
