<!--
SPDX-FileCopyrightText: Copyright (c) Meta Platforms, Inc. and affiliates.
SPDX-FileCopyrightText: © 2024 Rivos Inc.

SPDX-License-Identifier: Apache-2.0
-->
# CLAUDE.md

Project-specific guidance for working in the OpenSIPI repository. This supplements
the global guidelines in `~/.claude/CLAUDE.md`.

## What this project is

OpenSIPI is an open-source, Python 3.10+ platform that automates **signal integrity
(SI)** and **power integrity (PI)** extractions for PCB/package designs. It reads
tabular input describing simulations, generates tool-specific scripts, drives a
commercial EDA solver to run the extractions, post-processes the results
(S-parameters / DCR), and produces a PDF/HTML report.

OpenSIPI itself is free and open source, but it orchestrates **commercial back-end
solvers** — currently only **Cadence Sigrity** (PowerSI, Clarity, PowerDC). The
solvers and their licenses are NOT included; nothing in this repo actually runs an
extraction without a licensed solver installed.

Four extraction types are supported, each mapping to a Sigrity solver:

| Extraction type | Solver        | Purpose                          |
| --------------- | ------------- | -------------------------------- |
| `PDN`           | PowerSI       | Power delivery network (Z-param) |
| `LSIO`          | PowerSI       | Low-speed IO (S-param)           |
| `HSIO`          | Clarity (FEM) | High-speed IO (S-param)          |
| `DCR`           | PowerDC       | DC resistance                    |

## Three-layer architecture

The codebase is organized as three conceptual layers (see `docs/Home/`):

1. **Front-end file I/O** — reads simulation input from CSV files or Google Sheets,
   writes results as touchstone (`.sNp`) / CSV and a PDF/HTML report.
2. **Mid-layer platform** — the OpenSIPI package itself (orchestration, parsing,
   post-processing, reporting).
3. **Back-end solvers** — external commercial EDA tools, driven via generated
   scripts (Tcl for Sigrity).

## Key modules (`opensipi/`)

- `integrated_flows.py` — top-level entry points users call: `sim2report()` (local)
  and `sim2report_gsuites()` (Google Sheets in / Google Drive out). Start here to
  understand the end-to-end flow.
- `sipi_infra.py` — `Platform` class, the central orchestrator. Builds the run
  folder structure, reads input, dispatches to the right solver executor, runs
  post-processing, and generates reports. This is the spine of the application.
- `file_in.py` — `FileIn`: parses CSV / Google Sheet input into the internal
  `input_data` dict (sim_input, stackup, settings, spec types).
- `sigrity_exec.py` — solver "executor" classes that drive a run:
  `PowersiPdnExec` (base) → `PowersiIOExec` → `ClarityExec`, plus `PowerdcExec`.
- `sigrity_tools.py` — solver "modeler" classes that generate Tcl and build the
  simulation models: `SpdModeler` (base) → `PowersiPdnModeler` →
  `PowersiIOModeler` → `ClarityModeler`, plus `PowerdcModeler`. Largest module;
  contains the bulk of the SI/PI domain logic (ports, nets, stackup, solder, etc.).
- `touchstone.py` — `TouchStone`: S-parameter post-processing (IL, RL, TDR,
  mixed-mode) and plot generation, built on scikit-rf.
- `gsheet_io.py` / `gdrive_io.py` — Google Sheets input and Google Drive result
  upload (used by the `_gsuites` flow).
- `constants/CONSTANTS.py` — input column titles, `SPEC_TYPE` definitions
  (frequency ranges + post-process keys), folder names. Central place for the
  vocabulary the input files use.
- `templates/` — Tcl templates (`temp_*.tcl`, `proc_common.tcl`) rendered to drive
  Sigrity, and HTML/PDF report templates (`reports/`, `temp_report.py`).
- `util/` — `common.py` (path/CSV/YAML helpers, the `SL` path-separator constant),
  `exceptions.py` (domain exceptions), `logs.py` (per-run logger).
- `autopwt/` — "auto power tree" — a separate Tkinter GUI utility
  (`autoPWT_GUI.py`). Largely independent of the main extraction flow; note some
  files here carry a Google LLC copyright header rather than Rivos.

## How a run flows

`sim2report(input_info, mntr_info)` →
1. `Platform(input_info)` — creates the on-disk run folder tree (`Dsn/`,
   `Xtract/Run_<timestamp>/` with `LocalDsn`, `LocalScript`, `SimFile`, `Result`,
   `Report`, `Log`, ...) and reads input.
2. `pf.drop_dsn_file()` — interactively prompts the user to place the design file
   (`.brd`, `.spd`, ODB++, `.mcm`) and confirm at the terminal.
3. `pf.parser()` — parses input and selects the solver executor by extraction type.
4. `pf.run()` — generates scripts, builds models ("Model Check"), runs the solver
   ("Model Run"), collects results. `.done` marker files track completed keys so a
   run can resume without redoing finished simulations.
5. `pf.report()` — post-processes and emits the report.

`docs/Home/Mid-layer-Platform.md` has the authoritative narrative of this workflow
and the folder structure; read it before changing run orchestration.

## Input format

Input is a set of CSV sheets (or Google Sheet tabs) in a folder. Mandatory sheets:
`Sim*` (per-simulation port/net definitions), `Stackup_Materials`,
`Special_Settings`; optional `Spec_Type`. The schemas are detailed and exact —
`docs/Home/Front-end-Files-IO.md` is the reference. `examples/Olympus/` is a full
worked example (input CSVs, launch scripts, and sample output reports).

## Dev workflow & conventions

- **Environment:** Poetry. `poetry install`, then `poetry shell`. The project
  targets Python `^3.10`.
- **Formatting/linting:** enforced by pre-commit — `black` (line length 100),
  `isort` (black profile), `flake8`, `flynt`, `pyupgrade`, `prettier` (YAML).
  Run `pre-commit run` before committing.
- **Licensing:** every file must carry an SPDX header (Apache-2.0). Managed by
  `reuse` — run `reuse lint` after adding files. There is **no CI test suite**;
  `tests/` is gitignored and not part of the repo. Verify changes by running the
  example flow or targeted manual checks.
- **Contribution model:** fork → PR (this is a public GitHub project under
  `rivosinc/opensipi`). See `CONTRIBUTING.md`.
- **Versioning:** bump `version` in `pyproject.toml` AND `__version__` in
  `opensipi/__init__.py` together — they must stay in sync.

## Working in this codebase

- Match the existing style: classes use an executor/modeler inheritance hierarchy;
  prefer extending the right base class over duplicating logic.
- Paths are built with the `SL` separator constant from `util/common.py` for
  cross-platform (Windows/Linux) support — don't hardcode `/` or `\`.
- The solver back-ends cannot run here without licensed Cadence Sigrity tools, so
  end-to-end extraction can't be executed in this environment. Reason about
  correctness from the generated Tcl, the input parsing, and post-processing logic.
- Active development focus (per recent commits) is on port/net handling in
  `sigrity_tools.py` (e.g. differential ports, nearby-ground-node detection).
