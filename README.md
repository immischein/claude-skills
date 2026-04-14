# claude_skills

A collection of Claude Code skills for scientific Python development.

## Skills

### [sci-plot](sci-plot.md) — publication-quality figures & professional curve fits

Guides an end-to-end workflow for creating thesis- or paper-ready matplotlib figures:

- Asks the right questions upfront before writing any code (axes, fit model, exponent convention, output format)
- Detects the appropriate fit model from context (power law, exponential, Gaussian, linear, log, survival, sinusoid, Arrhenius)
- Drops in ready-to-paste **interactive fit-range sliders** (single dataset and multi-dataset sequential)
- Enforces explicit exponent-convention declaration before writing any fit code — the single biggest source of wasted iterations
- Applies a consistent serif style with a colorblind-safe 8-color palette and transparent backgrounds
- Handles common problems: noisy scatter, legend overflow, colorbar misalignment, OOM on large datasets, periodic artifacts
- Propagates fit errors to derived quantities
- Exports transparent PDF/PNG with correct `dpi`, `bbox_inches`, and `transparent` settings

**Triggers:** `plot`, `figure`, `fit`, `thesis`, `publication`, `log-log`, `savefig`, `transparent`, `slider`, `ipywidgets`, `power law`, `scaling`, `exponent`, `noisy`, `artifacts`, `colormap`, `legend`, `axis`, `matplotlib`

---

### [restructure](restructure.md) — split a module into compute script + analysis notebook

Restructures any Python module into two clean parts:

- A **compute script** — pure data generation, no plotting, saves to `data/` as `.npz`/`.pkl`, runnable from any working directory
- An **analysis notebook** — loads from `data/`, interactive fit-range sliders, thesis-quality transparent PDF output

**Triggers:** `/restructure <module-name>`

---

## Installation

### Install a single skill

```sh
# sci-plot
curl -L https://raw.githubusercontent.com/immischein/claude_skills/master/sci-plot.md \
  -o ~/.claude/commands/sci-plot.md

# restructure
curl -L https://raw.githubusercontent.com/immischein/claude_skills/master/restructure.md \
  -o ~/.claude/commands/restructure.md
```

### Install all skills at once

```sh
for skill in sci-plot restructure; do
  curl -L https://raw.githubusercontent.com/immischein/claude_skills/master/${skill}.md \
    -o ~/.claude/commands/${skill}.md
done
```

> **Note:** `sci-plot.md` is installed as `sci-plot.md` so the slash command becomes `/sci-plot`.

### Project-local install (one project only)

```sh
mkdir -p .claude/commands

curl -L https://raw.githubusercontent.com/immischein/claude_skills/master/sci-plot.md \
  -o .claude/commands/sci-plot.md

curl -L https://raw.githubusercontent.com/immischein/claude_skills/master/restructure.md \
  -o .claude/commands/restructure.md
```

### Using the skills

Once installed, open Claude Code and either:

- Type `/sci-plot` or `/restructure <module>` to invoke explicitly
- Or just mention a trigger word — Claude will load the relevant skill automatically

## Requirements

- Python 3.8+
- `matplotlib`, `numpy`, `scipy`
- `ipywidgets` (for interactive sliders, Jupyter only)

## License

MIT — see [LICENSE](LICENSE). If you adapt these skills publicly, a mention or link back is appreciated.
