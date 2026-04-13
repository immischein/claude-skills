# sci-plot

A Claude Code skill for publication-quality scientific figures and professional curve fits in Python/Jupyter with matplotlib.

## What it does

Guides an end-to-end workflow for creating thesis- or paper-ready figures:

- Asks the right questions upfront before writing any code (axes, fit model, exponent convention, output format)
- Detects the appropriate fit model from context (power law, exponential, Gaussian, linear, log, survival, sinusoid, Arrhenius)
- Drops in ready-to-paste **interactive fit-range sliders** (single dataset and multi-dataset sequential)
- Enforces explicit exponent-convention declaration before writing any fit code — the single biggest source of wasted iterations
- Applies a consistent serif style with a colorblind-safe 8-color palette and transparent backgrounds
- Handles common problems: noisy scatter, legend overflow, colorbar misalignment, OOM on large datasets, periodic artifacts
- Propagates fit errors to derived quantities
- Exports transparent PDF/PNG with correct `dpi`, `bbox_inches`, and `transparent` settings

## Triggers

The skill activates on any of:

`plot`, `figure`, `fit`, `thesis`, `publication`, `log-log`, `savefig`, `transparent`,
`slider`, `ipywidgets`, `power law`, `scaling`, `exponent`, `noisy`, `artifacts`,
`colormap`, `legend`, `axis`, `matplotlib`

## Installation

### Option 1 — one-liner (recommended)

Downloads the skill directly into your Claude Code commands directory:

```sh
curl -L https://raw.githubusercontent.com/immischein/sci-plot/master/skill.md \
  -o ~/.claude/commands/sci-plot.md
```

### Option 2 — clone and copy

```sh
git clone https://github.com/immischein/sci-plot.git
cp sci-plot/skill.md ~/.claude/commands/sci-plot.md
```

### Option 3 — project-local only

If you only want the skill available in one project:

```sh
mkdir -p .claude/commands
curl -L https://raw.githubusercontent.com/immischein/sci-plot/master/skill.md \
  -o .claude/commands/sci-plot.md
```

### Using it

Once installed, open Claude Code in any project and either:

- Type `/sci-plot` to invoke it explicitly
- Or just ask Claude to plot or fit something — the trigger words above will load it automatically

## Before / after

**Without the skill** — loose, back-and-forth session:

```
User:   "Add a power-law fit to my data."
Claude: [writes code with hardcoded x range, picks an arbitrary exponent convention, no slider]
User:   "The range is wrong."
Claude: [changes range, still no interactive control, convention still unclear]
...  (3–4 more rounds)
```

**With the skill** — structured from the start:

```
User:   "Add a power-law fit to my data."
Claude: "Before I write the fit, a few quick questions:
         1. What are x and y — what do they represent?
         2. Fitting y = A·x^n: confirm the exponent of interest is n itself,
            not a function of n (e.g. Q = 2/n)?
         3. Any zeros or NaN in the data at early values?
         4. Output: inline display or transparent PDF?"

User:   [answers]

Claude: [drops in universal interactive fitter with FloatRangeSlider,
         states the exact convention above the code block,
         uses colorblind-safe palette and serif rcParams]
```

## Requirements

- Python 3.8+
- `matplotlib`, `numpy`, `scipy`
- `ipywidgets` (for interactive sliders, Jupyter only)

## License

MIT — see [LICENSE](LICENSE). If you adapt this skill publicly, a mention or link back is appreciated.
