# Photonics Designs

Private repository containing all PIC design work: GDSFactory scripts, Jupyter notebooks, simulation results, plots, and reports.

**This repo is the companion to [photonics-workstation](https://github.com/pic-asso/photonics-workstation),** which provides the shared infrastructure (setup, tools, Claude AI slash commands). The two repos must be cloned side by side for the CLAUDE.md import to work.

## Prerequisites

The `photonics-workstation` template must be cloned at `../photonics-workstation/` relative to this repo — i.e. both repos share the same parent directory:

```
~/Desktop/
├── photonics-workstation/   ← infrastructure template (must exist here)
└── photonics-designs/       ← this repo
```

If you haven't set up the workstation yet, do that first:

```bash
git clone git@github.com:pic-asso/photonics-workstation.git ~/Desktop/photonics-workstation
cd ~/Desktop/photonics-workstation
chmod +x setup.sh && ./setup.sh && source ~/.bashrc
conda env create -f environment.yml
tidy3d configure
```

## Cloning this repo

```bash
git clone git@github.com:<your-org>/photonics-designs.git ~/Desktop/photonics-designs
```

That's it. All Claude AI behavioural rules (DRC discipline, simulation cost gates, NDA file guards) are inherited automatically from `photonics-workstation/CLAUDE.md` via the one-line import in `CLAUDE.md`. No extra config needed.

## Opening Claude Code

Always open Claude Code from this directory, not from `photonics-workstation/`:

```bash
cd ~/Desktop/photonics-designs
claude
```

All `/slash-commands` are globally installed by `setup.sh` and will be available here.

## Repository Structure

```
photonics-designs/
│
├── CLAUDE.md                   # Imports rules from ../photonics-workstation/CLAUDE.md
│
├── designs/                    # GDSFactory component scripts and extracted parameters (tracked)
│   └── [component_name]/
│       ├── [component_name].py
│       └── params_[ComponentName]_YYYYMMDD_HHMM.json
│
├── notebooks/                  # Jupyter design notebooks (tracked)
│
├── layout/                     # GDS/OAS layouts (git-ignored — binary)
│   └── [component_name]/
│       └── [ComponentName]_YYYYMMDD_HHMM.gds
│
├── pdk-info/                   # Foundry PDK documents (git-ignored — NDA)
├── drc-rules/                  # Extracted YAML rule decks (git-ignored — NDA)
├── tapeout/                    # MPW submission packages (git-ignored — NDA)
│
└── analysis/
    ├── simulation/             # Raw simulation outputs (git-ignored — large binary)
    │   └── [component_name]/
    ├── plots/                  # Generated figures — PNG/PDF (tracked)
    │   └── [component_name]/
    └── reports/                # KPI CSVs, corner results, MD/LaTeX reports (tracked)
        └── [component_name]/
```

### Naming conventions

All outputs use `[component_name]` (snake_case) subfolders so `/generate-report` can discover every artefact with a single glob per type.

| Output type | Path | Naming pattern |
|---|---|---|
| Design scripts | `designs/[component_name]/` | `[component_name].py` |
| Extracted parameters | `designs/[component_name]/` | `params_[ComponentName]_YYYYMMDD_HHMM.json` |
| GDS layouts | `layout/[component_name]/` | `[ComponentName]_YYYYMMDD_HHMM.gds` |
| DRC markers | `layout/[component_name]/` | `[ComponentName]_YYYYMMDD_HHMM.lyrdb` |
| Raw sim data | `analysis/simulation/[component_name]/` | `sim_[ComponentName]_[Timestamp].[ext]` |
| Plots | `analysis/plots/[component_name]/` | `plot_[ComponentName]_[Timestamp].png` |
| KPI / corner CSVs | `analysis/reports/[component_name]/` | `[type]_[ComponentName]_[Timestamp].csv` |
| Reports (MD + TeX) | `analysis/reports/[component_name]/` | `report_[ComponentName]_YYYYMMDD_HHMM.{md,tex}` |

## What is and isn't tracked

| Path | Tracked | Reason |
|---|---|---|
| `designs/` | Yes | Core IP — design scripts |
| `notebooks/` | Yes | Exploration and analysis |
| `analysis/plots/` | Yes | Publication figures |
| `analysis/reports/` | Yes | KPI tables and executive reports |
| `layout/` | No | Binary GDS/OAS files |
| `analysis/simulation/` | No | Large HDF5/NumPy archives |
| `pdk-info/` | No | NDA — foundry PDK |
| `drc-rules/` | No | NDA — derived from foundry PDF |
| `tapeout/` | No | NDA — MPW submission |

## Updating the workstation

When the template gets improvements (new slash commands, updated utility scripts):

```bash
cd ~/Desktop/photonics-workstation
git pull && ./setup.sh
```

No action needed here — the CLAUDE.md import resolves live and the new commands are immediately available.
