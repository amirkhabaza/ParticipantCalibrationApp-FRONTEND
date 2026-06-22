# Participant Calibration App

A Python application for running **9-point eye-tracker calibration** sessions. It displays fixation targets on screen and records **when** and **where** each target appeared, so gaze data from a separate eye tracker can be aligned later.

Built for NASA internship eye-tracking research. Tested on **macOS** and **Windows 11** (HP EliteBook).

---

## What this project does (in plain language)

During eye-tracker calibration, a participant must look at known points on the screen while the tracker records where their eyes are pointing. This app handles the **stimulus side** of that process:

1. Opens a fullscreen black window on the participant monitor.
2. Shows instructions, then waits for the participant to press **SPACEBAR**.
3. Presents **9 calibration dots** one at a time in a random (but repeatable) order.
4. Saves a **CSV file** with the exact screen position and timestamps for each dot.

This app does **not** connect to the eye tracker directly. The eye tracker runs in its own software. After the session, you combine:

- **Gaze data** from the eye tracker (timestamps + gaze coordinates)
- **`calibration_targets.csv`** from this app (timestamps + known target positions)

…to build or validate the calibration mapping.

---

## How it fits in the full pipeline

```
┌─────────────────────┐     ┌──────────────────────────┐
│  Eye tracker        │     │  This app                │
│  (separate software)│     │  calibration_9point.py   │
│                     │     │                          │
│  Records gaze (x,y) │     │  Shows dots at known     │
│  with timestamps    │     │  (x,y) with timestamps   │
└─────────┬───────────┘     └────────────┬─────────────┘
          │                              │
          └──────────┬───────────────────┘
                     ▼
          ┌──────────────────────┐
          │  Analysis / backend  │
          │  Match gaze samples  │
          │  to target windows   │
          │  → calibration model │
          └──────────────────────┘
```

---

## Project structure

```
ParticipantCalibrationApp-FRONTEND-/
├── README.md                      ← This file
├── Frontend/
│   ├── calibration_9point.py    ← Main application (run this)
│   └── requirements.txt           ← Python dependencies (PsychoPy)
└── output/
    └── calibration_targets.csv    ← Created after each run
```

| File | Purpose |
|------|---------|
| `Frontend/calibration_9point.py` | Fullscreen calibration stimulus + CSV logging |
| `Frontend/requirements.txt` | Lists `psychopy` as the only pip dependency |
| `output/calibration_targets.csv` | Ground-truth target log (auto-created on run) |

---

## Session flow (what the participant sees)

```
[Instructions screen]
   "Focus on each dot... Press SPACEBAR to begin"
              │
              ▼ SPACEBAR
[0.3s blank screen]
              │
              ▼
[Target 1 of 9]  shrinking bullseye for 1.0 second
              │
[0.3s blank]
              │
              ▼
         ... repeat for all 9 targets ...
              │
              ▼
[Calibration complete — press any key to exit]
```

**Controls during the session**

| Key | Action |
|-----|--------|
| **SPACEBAR** | Start calibration (instructions screen only) |
| **ESC** | Abort session (completed targets are still saved) |
| **Any key** | Exit after completion |

---

## The 9-point grid

Targets are placed on a **3×3 grid** at 10%, 50%, and 90% of screen width and height (10% inset from each edge).

**Target IDs** are fixed by position (row-major, top-left → bottom-right):

```
┌─────┬─────┬─────┐
│  1  │  2  │  3  │   ← top row
├─────┼─────┼─────┤
│  4  │  5  │  6  │   ← middle row
├─────┼─────┼─────┤
│  7  │  8  │  9  │   ← bottom row
└─────┴─────┴─────┘
```

**Presentation order** is shuffled randomly, but always the same shuffle because `RANDOM_SEED = 42`. Example order: `4 → 7 → 8 → 5 → 9 → 3 → 6 → 1 → 2`.

The ID in the CSV always refers to the **grid position**, not the order the dot was shown.

---

## The bullseye stimulus

Each target is a white-on-black **bullseye**:

- Center **dot** (4 px radius)
- **Crosshairs** (24 px arms)
- **Shrinking ring** (starts at 48 px, shrinks to ~6 px over 1 second)

The ring animation gives participants a clear fixation point and a sense of timing.

---

## Output CSV format

After each run, results are saved to:

```
output/calibration_targets.csv
```

| Column | Description |
|--------|-------------|
| `Timestamp_Start` | Unix time (seconds) when the target appeared |
| `Timestamp_End` | Unix time (seconds) when the target disappeared |
| `Target_ID` | Grid ID 1–9 (see grid diagram above) |
| `Target_X_Px` | Horizontal pixel position (top-left origin) |
| `Target_Y_Px` | Vertical pixel position (top-left origin) |
| `Screen_Width` | Monitor width in pixels |
| `Screen_Height` | Monitor height in pixels |

**Example row:**

```csv
Timestamp_Start,Timestamp_End,Target_ID,Target_X_Px,Target_Y_Px,Screen_Width,Screen_Height
1782069597.526124,1782069598.538956,4,151,491,1512,982
```

This means target **4** (middle-left) was shown from timestamp `1782069597.53` to `1782069598.54` at pixel `(151, 491)` on a `1512×982` screen.

Use `Timestamp_Start` and `Timestamp_End` to find which gaze samples occurred while the participant should have been looking at that target.

---

## Key configuration (inside the script)

These constants at the top of `calibration_9point.py` control behavior:

| Constant | Default | Meaning |
|----------|---------|---------|
| `TARGET_DURATION_S` | `1.0` | How long each dot is shown (seconds) |
| `RANDOM_SEED` | `42` | Shuffle seed (same order every run) |
| `PRE_TARGET_BLANK_S` | `0.3` | Blank screen before first target |
| `INTER_TARGET_BLANK_S` | `0.3` | Blank screen between targets |
| `EDGE_INSET_FRACTION` | `0.10` | Grid inset from screen edges (10%) |

---

## Requirements

- **Python 3.10** (recommended; PsychoPy pip wheels support 3.8–3.10)
- **PsychoPy** (`pip install psychopy`)
- A monitor with **OpenGL** support (standard on Mac and HP EliteBook)
- Run on the **same monitor** used for eye tracking (`screen=0`, primary display)

---

## How to run

### Standard setup (Mac or Windows, with admin / normal install)

```bash
# 1. Clone the repo
git clone https://github.com/amirkhabaza/ParticipantCalibrationApp-FRONTEND-.git
cd ParticipantCalibrationApp-FRONTEND-/Frontend

# 2. Create a virtual environment (Python 3.10)
python3.10 -m venv .venv

# 3. Activate it
source .venv/bin/activate        # macOS / Linux
# .venv\Scripts\activate         # Windows (PowerShell)

# 4. Install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# 5. Run the calibration
python calibration_9point.py
```

### Windows (HP EliteBook) — PowerShell

```powershell
cd Frontend
py -3.10 -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements.txt
python calibration_9point.py
```

### Mac without admin access (no Homebrew, no sudo)

Use **uv** — installs Python 3.10 entirely in your home folder:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

uv python install 3.10
cd Frontend
uv venv --python 3.10 .venv
source .venv/bin/activate
uv pip install -r requirements.txt
python calibration_9point.py
```

### Auto mode (skip instructions and exit prompt)

Useful for testing without keyboard input:

```bash
python calibration_9point.py --auto
```

---

## Platform-specific notes

| Platform | Notes |
|----------|-------|
| **macOS** | Retina displays are handled automatically (`useRetina` enabled on Darwin) |
| **Windows** | DPI scaling is handled for displays at 125%/150% (HP EliteBook) |
| **Both** | Use fullscreen on the eye-tracker monitor; avoid running on a secondary display unless configured |

---

## Main code sections (for explaining the script)

| Function | What it does |
|----------|--------------|
| `generate_grid_targets()` | Computes 9 screen positions from width/height |
| `build_bullseye_stimuli()` | Creates dot, crosshairs, and ring PsychoPy objects |
| `present_shrinking_bullseye()` | Animates and displays one target for 1 second |
| `resolve_window_size()` | Gets correct pixel dimensions (handles Mac Retina) |
| `save_calibration_csv()` | Writes results to `output/calibration_targets.csv` |
| `run_calibration()` | Orchestrates the full session from window open to CSV save |

Entry point: `main()` → `run_calibration()`.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---------|--------------|-----|
| `python3.10: command not found` | Python 3.10 not installed | Install 3.10 via python.org, Homebrew, or `uv` (see above) |
| Only Python 3.13 available | Newer macOS default | Use `uv python install 3.10` (no admin needed) |
| `pip install psychopy` fails | Wrong Python version | Use Python 3.10, not 3.11+ |
| Black screen / no window | OpenGL or display issue | Update graphics drivers; run natively (not remote desktop) |
| Wrong coordinates in CSV | Display scaling | Run on primary monitor; on Windows ensure DPI awareness is active |
| CSV missing some rows | Session aborted with ESC | Partial data is still saved for completed targets |
| `ModuleNotFoundError: psychopy` | venv not activated | Run `source .venv/bin/activate` before `python` |

---

## What this app does NOT do

- Does not read or control eye-tracker hardware
- Does not compute the calibration mapping (that is backend/analysis work)
- Does not upload data anywhere (CSV is saved locally only)
- Does not randomize target positions (only presentation order is shuffled)

---

## Quick explanation script (for presenting to someone)

> "This is a PsychoPy app that runs a standard 9-point eye-tracker calibration. It fullscreen-opens on the participant monitor, shows nine fixation dots in random order, and logs exactly when and where each dot appeared. The eye tracker records gaze separately. After the session, we align the gaze timestamps with this CSV to calibrate the tracker. The whole thing is one Python script plus a PsychoPy dependency, and it outputs a CSV to the `output/` folder."

---

## License / repo

GitHub: [ParticipantCalibrationApp-FRONTEND-](https://github.com/amirkhabaza/ParticipantCalibrationApp-FRONTEND-)
