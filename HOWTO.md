# How to Work Effectively with Claude on PIC Designs

This guide covers how to get accurate, reliable results when running the full design workflow
— from paper to layout to simulation to report — using the Claude AI slash commands in this
workstation. Read it before starting your first design.

---

## The Golden Rule: Plan Before You Generate

The most common failure mode is moving too fast. Each stage of the workflow feeds the next,
so an unnoticed error early on (a wrong unit, a misread coupling length) will propagate
silently through layout, simulation, and report. **Pause and verify at every handoff.**

The recommended sequence:

```
1. /extract-paper      → verify the JSON output carefully before proceeding
2.  ↓ confirm topology → describe the device in your own words, ask Claude to confirm
3. /paper-to-layout    → visual check of the GDS before any simulation
4.  ↓ lock objectives  → agree on metrics, wavelength range, pass/fail criteria
5. /component-sim      → review cost estimate, then approve
6. /corner-analysis    → only if nominal sim looks correct
7. /circuit-sim        → system-level validation
8. /drc-assistant      → Tier-1 during iteration, Tier-2 before tapeout
9. /generate-report    → only after all artefacts exist on disk
```

Never skip the confirmation steps between stages.

---

## Starting a Session

Always open Claude Code from `photonics-designs/`, not from `photonics-workstation/`:

```bash
cd ~/Desktop/photonics-designs
claude
```

**First time in a new clone:** Claude will show a one-time approval dialog to load the
`@../photonics-workstation/CLAUDE.md` import. Approve it. This gives Claude all the
behavioural rules (DRC discipline, simulation cost gates, file naming conventions, etc.).

**Brief Claude at the start of each session.** A short context message goes a long way:

> "I'm reproducing the bent directional coupler from [Author et al., 2023]. The target
> wavelength is 1550 nm. I've already extracted parameters — picking up from the layout stage."

Claude does not retain memory between sessions beyond what is in `MEMORY.md`. Any in-progress
state, decisions made, or partial results from a previous session should be re-stated.

---

## Where Claude Is Most Likely to Be Wrong

Be especially critical at these points:

| Risk | What to watch for |
|---|---|
| **Unit errors** | nm vs µm, especially for gaps and coupling lengths — papers are inconsistent |
| **Radius vs diameter** | papers use both; always check which one the formula uses |
| **Parameter ambiguity** | "width" — waveguide core width or total rib width? Ask explicitly |
| **Topology assumptions** | anything not shown in the paper will be inferred — sometimes wrongly |
| **Simulation defaults** | mesh resolution, PML padding, runtime — reasonable guesses, not optimal values |
| **Port naming** | GDSFactory uses `o1`, `o2`… — easy to get backwards, check against the paper figure |
| **Missing parameters** | if a value has no verbatim source quote in the JSON, Claude inferred it |

The `/extract-paper` JSON includes a `source_quote` field for every value. **If a value has
no quote, Claude invented it.** Challenge those before proceeding.

---

## Command-by-Command Tips

### `/extract-paper`

- Provide the full PDF if possible, not just a screenshot.
- Point Claude to specific figures and tables explicitly:
  *"Dimensions are in Table 1 and Fig. 3b. The fabrication tolerances are in Section III."*
- After receiving the JSON, read every value and verify the units.
- Ask Claude to list any values it was uncertain about or had to infer.
- Do not pass the JSON to `/paper-to-layout` until it is correct.

### `/paper-to-layout`

Provide all three sources together:

1. **The verified parameter JSON** from `/extract-paper`
2. **The paper itself** (for context Claude may have missed)
3. **Layout or schematic figures from the paper** — this is the most underused input

The JSON gives Claude *numbers*. The figures give Claude *topology*: which port is the
through port, whether there is an S-bend input, whether the coupler is symmetric, where tapers
start and end, how the device connects to the rest of the circuit. A figure eliminates entire
categories of geometric assumption.

Useful figure types to provide:
- Schematic diagrams (clearest for topology)
- SEM images (show fabricated geometry, reveal taper presence, confirm layer usage)
- Simulation field plots (confirm mode propagation direction)

Also tell Claude explicitly:
- Which foundry layer names to use (e.g. `WG`, `M1` if your PDK uses uppercase)
- Whether you need a standalone cell or a sub-cell to be assembled into a larger circuit

After receiving the GDSFactory script, **open the GDS in KLayout before approving**. Verify:
- Port positions and orientations match the paper figure
- Coupling region geometry looks correct
- No obvious dimensional errors (scale the ruler against the paper)

### `/component-sim`

State your objectives explicitly before Claude writes any code:

- *"I want S21 transmission from o1 to o2 over 1480–1620 nm with 5 nm steps."*
- *"Keep the FlexCredit cost under 15 credits."*
- *"Recommend whether 2D or 3D is appropriate for this geometry and explain why."*

Claude will recommend a solver — take that recommendation seriously. Running 3D FDTD on a
structure that could be handled by EME or FEM wastes credits and time. Ask for the reasoning
if it is not clear.

The simulation script will always include a cost estimate and a `y/n` gate before any cloud
submission. Do not bypass it.

### `/corner-analysis`

Only run this after the nominal simulation result looks physically correct. Corners on a
broken nominal case are meaningless and waste cloud credits.

Specify the fabrication tolerance assumptions explicitly — do not let Claude guess them:
- *"Use ±20 nm width variation and ±10 nm thickness variation based on our foundry spec."*

### `/drc-assistant`

Run `/extract-drc-rules` first if `drc-rules/foundry_rules.yaml` does not exist or is
outdated. Claude will not guess rule values — it reads from the YAML only.

- **Tier 1** (`drc-quick`): use during active design iteration. Fast, YAML-based.
- **Tier 2** (`drc-signoff`): use before tapeout. Full foundry deck via `.lydrc` file.

If a rule shows `null` in the YAML it means it was not extracted — not that it passes. Update
the YAML before assuming the design is compliant.

### `/generate-report`

Before invoking, verify that all expected artefacts exist in the right locations:

```
designs/[component_name].py                          ← design script
layout/[component_name]/[ComponentName]_*.gds        ← GDS layout
layout/[component_name]/[ComponentName]_*.lyrdb      ← DRC markers
analysis/simulation/[component_name]/sim_*.npz|h5    ← raw sim data
analysis/plots/[component_name]/plot_*.png           ← figures
analysis/reports/[component_name]/*_results_*.csv    ← KPI tables
```

Claude will list every file it finds and ask for confirmation before writing anything.
Missing sections get a visible `[No data found — run /command-name first]` placeholder
rather than fabricated content. This is intentional — do not ask Claude to fill them in.

---

## Providing Good Feedback Mid-Workflow

When Claude produces something wrong, be specific:

| Instead of... | Say... |
|---|---|
| "That's wrong" | "The gap should be 180 nm, not 180 µm — check Table 1 line 3" |
| "The layout looks off" | "The input waveguide should enter from the left with an S-bend, not straight — see Fig. 2a" |
| "The simulation is too slow" | "The domain is larger than needed — the coupler is 12 µm long, reduce X padding to 2 µm" |

Specific corrections train the current session. Vague corrections lead to guessing.

---

## Session Length and Context

Long sessions accumulate context but also accumulate drift — early decisions get buried and
Claude may start making assumptions inconsistent with what was agreed earlier. For a full
design flow, consider splitting across sessions:

| Session | Scope |
|---|---|
| 1 | `/extract-paper` → parameter review → `/paper-to-layout` → GDS visual check |
| 2 | `/component-sim` → result review → `/corner-analysis` |
| 3 | `/circuit-sim` → `/drc-assistant` → `/generate-report` |

At the start of each new session, paste the verified parameter JSON and briefly state where
you left off. This is faster than re-extracting and eliminates re-introduction of errors.

---

## FlexCredit Discipline

- Always run `check-credits` before starting a simulation-heavy session.
- Never approve a cloud job without reading the cost estimate.
- Corner sweeps submitted via `tidy3d.web.Batch` cost the sum of all corners — check that
  the per-corner cost times the number of corners is within budget before approving.
- Prefer EME (`/component-sim` with MEOW) for tapers and grating couplers — it is orders of
  magnitude cheaper than 3D FDTD for separable structures.

---

## File Naming and Organisation

All outputs must follow the per-design subfolder convention so `/generate-report` can find
them automatically:

| Output | Path | Pattern |
|---|---|---|
| Design script | `designs/` | `[component_name].py` |
| GDS | `layout/[component_name]/` | `[ComponentName]_YYYYMMDD_HHMM.gds` |
| DRC markers | `layout/[component_name]/` | `[ComponentName]_YYYYMMDD_HHMM.lyrdb` |
| Raw sim data | `analysis/simulation/[component_name]/` | `sim_[ComponentName]_[Timestamp].[ext]` |
| Plots | `analysis/plots/[component_name]/` | `plot_[ComponentName]_[Timestamp].png` |
| KPI CSVs | `analysis/reports/[component_name]/` | `[type]_[ComponentName]_[Timestamp].csv` |
| Reports | `analysis/reports/[component_name]/` | `report_[ComponentName]_YYYYMMDD_HHMM.{md,tex}` |

`component_name` = snake_case (e.g. `bent_dc`). `ComponentName` = PascalCase (e.g. `BentDC`).

If you save files with non-standard names, `/generate-report` will not find them and will
produce a report full of placeholders.
