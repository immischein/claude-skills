---
name: restructure
description: Restructure a module into a compute script + analysis notebook following the compute-script / analysis-notebook pattern
---

Restructure the module `$ARGUMENTS` into a compute script + analysis notebook by following these steps **in order**:

0. **Read `CLAUDE.md`** — ask the user: "Is there a `CLAUDE.md` with project conventions I should follow?" If yes, read it before doing anything else. If no, skip.

1. **Create `data/` subfolder** inside the module directory if it doesn't already exist.
2. **Create compute script** (`<module>/run_<name>.py` or `scripts/<name>_compute.py`) — pure data generation, no plotting, saves results to `data/` as `.npz` or `.pkl`. Use `os.path` relative to `__file__` for all paths so it is runnable from any working directory.
3. **Create analysis notebook** (`<module>/notebook_<name>.ipynb`) — loads from `data/`, performs fits with an interactive ipywidgets range selector (FloatRangeSlider in log-space + live matplotlib redraw), produces thesis-quality plots saved as transparent-background PDFs.
4. **Update imports and paths** in both files.
5. **Update or create `CLAUDE.md`** — ask the user: "Should I update (or create) a `CLAUDE.md` to document the new structure?" If yes: update it to reflect the new file structure, data format, and how to run the compute script and notebook. If no `CLAUDE.md` exists and the user says yes, create one. If the user says no, skip.
6. **Verify** both files run without errors (compute script end-to-end, notebook top-to-bottom).

If `$ARGUMENTS` is empty, ask the user which module to restructure before proceeding.
