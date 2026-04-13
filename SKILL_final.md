---
name: sci-plot
description: End-to-end guide for publication-quality scientific plots in Python/Jupyter with matplotlib. Use this skill whenever the user wants to: create or fix a matplotlib figure, apply a thesis/paper style, add any fit (power-law, exponential, linear, Gaussian), add interactive fit-range sliders, fit multiple datasets sequentially, do a scaling collapse, compute local exponents, troubleshoot noisy/artifacty/slow plots, fix legends/colorbars/axes, or export transparent PDF/PNG. Trigger on: "plot", "figure", "fit", "thesis", "publication", "log-log", "savefig", "transparent", "slider", "ipywidgets", "power law", "scaling", "exponent", "noisy", "artifacts", "colormap", "legend", "axis", "matplotlib".
---

# Scientific Plot Skill

Hard-won lessons from building thesis-quality physics/math figures. Covers style, data cleaning, all fit modes, interactivity, and export.

---

## Step 0 — Ask upfront (one message, not after rejection)

Before writing any code, ask:

1. What are x and y, and what do they represent?
2. Plot type: scatter, line, histogram, heatmap, bar, log-log, semilog?
3. Fit needed? If yes: **infer the model from context first** (see Model Detection below), state your guess, and ask the user to confirm or correct — do not ask them to name the model from scratch.
4. **State the exponent convention explicitly** — e.g. `y ~ x^α` vs `y ~ x^(−α)` vs `y ~ x^(1−β)`. Ask if unsure — this is the single biggest source of wasted rounds.
5. Known data issues: NaN/inf, zeros to exclude, periodic artifacts, edge effects?
6. Multiple datasets to fit? If yes, use the sequential multi-dataset slider (see below).
7. Output: transparent PDF/PNG for thesis/paper, or inline display?
8. Other notebooks with the same figure type that should get the same treatment?

---

## Model Detection

Before asking the user which fit to use, infer it from context. Check in order:

| Signal | Inferred model |
|--------|---------------|
| log-log axes requested / described | power law `y = A·x^n` |
| semilog-y axes / "exponential decay" / "lifetime" / "half-life" | exponential `y = A·exp(b·x)` |
| semilog-x axes / "per decade" / "log scale x only" | log fit `y = a·ln(x) + b` |
| linear axes + "proportional" / "rate" / "slope" | linear `y = m·x + c` |
| bell-shaped / "peak" / "distribution" / "resonance" | Gaussian `y = A·exp(−(x−μ)²/2σ²)` |
| "cumulative" / "survival" / "CDF" / power-law distributed samples | survival power law `P(X>x) ~ x^(−α)` |
| "level spacing" / "discrete spacing" / `p(s)` | `p(s) = A·s^(1−β)·exp(−c·s^(2−β))` or pure power law at small s |
| oscillating / periodic data | sinusoid `y = A·sin(ωx + φ) + c` |
| "Arrhenius" / activation energy / temperature | `ln(y) = −Ea/(kT) + const` → linear in `1/T` |

If the model is ambiguous, propose the top candidate and show the others as alternatives: *"I'll use a power law since your axes are log-log. If it's actually an exponential, switch the model dropdown."*

**Always use the universal interactive fitter** (below) when adding any fit to a notebook — it lets the user switch models without editing code.

---

## Universal Interactive Fitter

Drop this cell into any notebook. It auto-detects a sensible initial model from the axis scale, and lets the user switch model and fit range with dropdowns and a slider.

```python
import numpy as np
import matplotlib.pyplot as plt
import ipywidgets as widgets
from IPython.display import display, clear_output
from scipy.optimize import curve_fit

# ── USER CONFIG ──────────────────────────────────────────────
x_data = x_c        # your cleaned x array (no NaN, no zeros for log fits)
y_data = y_c        # your cleaned y array
x_label = 'x'
y_label = 'y'
# ─────────────────────────────────────────────────────────────

MODELS = {
    'Power law:  y = A·xⁿ':          lambda x, lA, n: lA + n*x,            # fit in log-space
    'Exponential:  y = A·eᵇˣ':       None,                                   # fit in linear space
    'Linear:  y = m·x + c':          None,
    'Gaussian:  y = A·exp(−(x−μ)²/2σ²)': None,
    'Log fit:  y = a·ln(x) + b':     None,
}

def do_fit(model_name, x, y, x_lo, x_hi):
    mask = (x >= x_lo) & (x <= x_hi) & np.isfinite(x) & np.isfinite(y)
    xm, ym = x[mask], y[mask]
    if len(xm) < 3:
        return None

    try:
        if model_name.startswith('Power law'):
            valid = (xm > 0) & (ym > 0)
            popt, pcov = curve_fit(lambda lx, lA, n: lA + n*lx,
                                   np.log(xm[valid]), np.log(ym[valid]))
            lA, n = popt
            A = np.exp(lA)
            n_err = np.sqrt(pcov[1, 1])
            xf = np.linspace(x_lo, x_hi, 300)
            yf = A * xf**n
            label = f'y = {A:.3g}·x^({n:.4f} ± {n_err:.4f})'
            return xf, yf, label, popt, pcov

        elif model_name.startswith('Exponential'):
            p0 = [ym.max(), (np.log(ym[-1]+1e-30) - np.log(ym[0]+1e-30)) / (xm[-1] - xm[0])]
            popt, pcov = curve_fit(lambda x, A, b: A * np.exp(b * x), xm, ym, p0=p0, maxfev=5000)
            A, b = popt
            xf = np.linspace(x_lo, x_hi, 300)
            yf = A * np.exp(b * xf)
            label = f'y = {A:.3g}·exp({b:.4f}·x)'
            return xf, yf, label, popt, pcov

        elif model_name.startswith('Linear'):
            popt, pcov = curve_fit(lambda x, m, c: m*x + c, xm, ym)
            m, c = popt
            xf = np.linspace(x_lo, x_hi, 300)
            yf = m * xf + c
            label = f'y = {m:.4f}·x + {c:.4g}'
            return xf, yf, label, popt, pcov

        elif model_name.startswith('Gaussian'):
            mu0 = xm[np.argmax(ym)]
            sig0 = (xm.max() - xm.min()) / 6
            popt, pcov = curve_fit(lambda x, A, mu, s: A*np.exp(-0.5*((x-mu)/s)**2),
                                   xm, ym, p0=[ym.max(), mu0, sig0], maxfev=5000)
            A, mu, sig = popt
            xf = np.linspace(x_lo, x_hi, 300)
            yf = A * np.exp(-0.5 * ((xf - mu) / sig)**2)
            label = f'A={A:.3g}, μ={mu:.4g}, σ={sig:.4g}'
            return xf, yf, label, popt, pcov

        elif model_name.startswith('Log fit'):
            valid = xm > 0
            popt, pcov = curve_fit(lambda x, a, b: a*np.log(x) + b, xm[valid], ym[valid])
            a, b = popt
            xf = np.linspace(max(x_lo, 1e-10), x_hi, 300)
            yf = a * np.log(xf) + b
            label = f'y = {a:.4f}·ln(x) + {b:.4g}'
            return xf, yf, label, popt, pcov

    except Exception as e:
        return None


# ── Widgets ───────────────────────────────────────────────────
model_dd = widgets.Dropdown(
    options=list(MODELS.keys()),
    description='Model:',
    style={'description_width': '60px'},
    layout=widgets.Layout(width='380px'),
)
use_log_x = widgets.Checkbox(value=(x_data.min() > 0), description='log x', layout=widgets.Layout(width='90px'))
use_log_y = widgets.Checkbox(value=(y_data.min() > 0), description='log y', layout=widgets.Layout(width='90px'))

x_min_safe = x_data[x_data > 0].min() if (x_data > 0).any() else x_data.min()
x_lo_init  = np.log10(x_min_safe) + 0.1 * (np.log10(x_data.max()) - np.log10(x_min_safe))
x_hi_init  = np.log10(x_min_safe) + 0.9 * (np.log10(x_data.max()) - np.log10(x_min_safe))

range_sl = widgets.FloatRangeSlider(
    value=[x_lo_init, x_hi_init],
    min=np.log10(x_min_safe), max=np.log10(x_data.max()),
    step=0.05, description='log₁₀(x):',
    style={'description_width': '80px'},
    layout=widgets.Layout(width='600px'),
    readout_format='.2f',
)
result_lbl = widgets.HTML(value='')
plot_out   = widgets.Output()


def redraw(change=None):
    x_lo = 10**range_sl.value[0]
    x_hi = 10**range_sl.value[1]
    fit  = do_fit(model_dd.value, x_data, y_data, x_lo, x_hi)

    with plot_out:
        clear_output(wait=True)
        fig, ax = plt.subplots(figsize=(9, 5))

        # Data
        plot_fn = ax.loglog if (use_log_x.value and use_log_y.value) else \
                  ax.semilogx if use_log_x.value else \
                  ax.semilogy if use_log_y.value else ax.plot
        plot_fn(x_data, y_data, '.', ms=3, alpha=0.5, color='#0072B2', label='Data')

        # Fit
        if fit:
            xf, yf, label, popt, pcov = fit
            plot_fn(xf, yf, '-', color='#D55E00', lw=2.5, label=label)
            ax.axvspan(x_lo, x_hi, color='#009E73', alpha=0.08)
            result_lbl.value = f'<b>{label}</b>'
        else:
            result_lbl.value = '<span style="color:red">Fit failed — try a different range or model</span>'

        ax.set_xlabel(x_label);  ax.set_ylabel(y_label)
        ax.legend(fontsize=9, frameon=False)
        plt.tight_layout();  plt.show()


for w in [model_dd, use_log_x, use_log_y, range_sl]:
    w.observe(redraw, names='value')

display(widgets.VBox([
    widgets.HBox([model_dd, use_log_x, use_log_y]),
    range_sl,
    result_lbl,
    plot_out,
]))
redraw()
```

After finding the right model and range, extract the final fit parameters:
```python
x_lo, x_hi = 10**range_sl.value[0], 10**range_sl.value[1]
fit = do_fit(model_dd.value, x_data, y_data, x_lo, x_hi)
xf, yf, label, popt, pcov = fit
print('Parameters:', popt)
print('Errors:    ', np.sqrt(np.diag(pcov)))
```

---

## Style

### Minimal serif (thesis/paper)
```python
plt.rcParams.update({
    'font.family': 'serif',
    'font.serif': ['DejaVu Serif', 'Georgia', 'Times New Roman'],
    'mathtext.fontset': 'dejavuserif',
    'axes.facecolor': 'none',
    'figure.facecolor': 'none',
    'axes.spines.top': False,
    'axes.spines.right': False,
    'axes.labelsize': 11,
    'xtick.labelsize': 9,
    'ytick.labelsize': 9,
    'xtick.direction': 'in',
    'ytick.direction': 'in',
    'xtick.major.size': 5,
    'ytick.major.size': 5,
    'legend.frameon': False,
    'legend.fontsize': 9,
    'agg.path.chunksize': 10000,  # prevents OOM on large scatter
})
```

### Colorblind-safe palette (8 colors, screen + print)
```python
PALETTE = ['#0072B2', '#D55E00', '#009E73', '#E69F00',
           '#CC79A7', '#56B4E9', '#F0E442', '#000000']
# Use: color=PALETTE[i % len(PALETTE)]
# Fit lines: '#D55E00' (orange), lw=2.5
# Reference/exact: '#7a7a7a' (gray), lw=1.5, ls='--'
# Fit window shading: '#009E73' (green), alpha=0.10
```

---

## Data Cleaning Before Plotting

Always filter before any log transform or fit:

```python
# Minimum: remove zeros and NaN
valid = (x > 0) & (y > 0) & np.isfinite(x) & np.isfinite(y)

# Periodic/parity artifact (e.g. data only defined on even steps)
valid = (t.astype(int) % 2 == 0) & (t >= 2) & (y > 0)

# Edge artifacts at small t
valid = (t >= 2) & (y > 0)

# Apply
x_c, y_c = x[valid], y[valid]
```

---

## Fit Techniques

### Rule: state the convention before writing fit code

Always write the exact relationship being fit before the code, e.g.:

> Fitting `f(x) = A · x^β` where derived quantity `Q = g(β)` — write the exact formula

> Fitting `y(t) = A · t^α` where derived quantity `Q = h(α)` — write the exact formula

> Fitting `p(s) = A · s^(1−β)` (NOT `s^β`) — clarify which exponent appears in the model

If the fit involves `s^(1−β)` vs `s^(−β)`, clarify before writing code. Getting this wrong silently gives a plausible-looking but wrong exponent.

---

### Power-law fit in log-space (core pattern)

Log-linearization is numerically superior to direct nonlinear fitting for power laws — it avoids poor initial guesses and works even when `A` spans many orders of magnitude.

```python
from scipy.optimize import curve_fit
import numpy as np

def linear_log(logx, logA, n):
    return logA + n * logx

# Fit window: set interactively (see slider section below)
mask = (x_c >= x_lo) & (x_c <= x_hi)
popt, pcov = curve_fit(linear_log, np.log(x_c[mask]), np.log(y_c[mask]))
logA, n = popt
A = np.exp(logA)
n_err = np.sqrt(pcov[1, 1])

# Fit quality: RMS relative residual
y_pred = A * x_c[mask]**n
residual = np.sqrt(np.mean(((y_c[mask] - y_pred) / y_c[mask])**2))
print(f'slope = {n:.4f} ± {n_err:.4f}   RMS residual = {residual*100:.1f}%')

# Plot
x_fit = np.logspace(np.log10(x_lo), np.log10(x_hi), 200)
ax.loglog(x_fit, A * x_fit**n, '-', color='#D55E00', lw=2.5,
          label=f'slope = {n:.3f} ± {n_err:.3f}')
ax.axvspan(x_lo, x_hi, color='#009E73', alpha=0.10)
```

---

### Error propagation to derived quantities

When the physical quantity is a function of the fit slope `n`, propagate the error using calculus:

```python
# Example: Q = -2 * n  →  Q_err = 2 * n_err
Q = -2.0 * n
Q_err = 2.0 * n_err

# Example: Q = 2 / n  →  Q_err = 2/n² * n_err  (quotient rule)
Q = 2.0 / n
Q_err = (2.0 / n**2) * n_err

# General: if Q = f(n), then Q_err ≈ |f'(n)| * n_err
```

---

### Fit results table (multi-dataset)

After fitting all datasets, print a summary table:

```python
print(f"{'label':>6}  {'x_lo':>7}  {'x_hi':>7}  {'Q':>8}  {'±':>2}  {'err':>7}")
print('-' * 55)
for label, res in fit_results.items():
    print(f"{label:>6}  {res['x_lo']:>7.1f}  {res['x_hi']:>7.1f}  "
          f"{res['Q']:>8.4f}  ±  {res['Q_err']:>7.4f}")
```

---

### Other fit models

```python
# Exponential: y = A * exp(b * x)
def exponential(x, A, b):
    return A * np.exp(b * x)
popt, pcov = curve_fit(exponential, x[mask], y[mask], p0=[y.max(), -0.1])
A, b = popt;  b_err = np.sqrt(pcov[1, 1])

# Linear
coeffs = np.polyfit(x[mask], y[mask], 1)
slope, intercept = coeffs

# Gaussian: y = A * exp(−(x−μ)²/(2σ²))
def gaussian(x, A, mu, sigma):
    return A * np.exp(-0.5 * ((x - mu) / sigma)**2)
p0 = [y.max(), x[np.argmax(y)], (x.max() - x.min()) / 6]
popt, pcov = curve_fit(gaussian, x[mask], y[mask], p0=p0)
A, mu, sigma = popt
```

---

## Interactive Fit Range Selector

**Never hardcode fit ranges.** Always use a slider.

### Single dataset

```python
import ipywidgets as widgets
from IPython.display import display, clear_output

log_min = np.log10(x_c.min())
log_max = np.log10(x_c.max())

slider = widgets.FloatRangeSlider(
    value=[log_min + 0.2*(log_max-log_min), log_min + 0.8*(log_max-log_min)],
    min=log_min, max=log_max, step=0.05,
    description='log₁₀(x):',
    style={'description_width': '80px'},
    layout=widgets.Layout(width='600px'),
    readout_format='.2f',
)
plot_out = widgets.Output()

def draw(change=None):
    x_lo, x_hi = 10**slider.value[0], 10**slider.value[1]
    with plot_out:
        clear_output(wait=True)
        fig, ax = plt.subplots(figsize=(8, 5))
        ax.loglog(x_c, y_c, '.', ms=3, alpha=0.5, color=PALETTE[0])
        mask = (x_c >= x_lo) & (x_c <= x_hi)
        if mask.sum() >= 3:
            popt, pcov = curve_fit(linear_log, np.log(x_c[mask]), np.log(y_c[mask]))
            n, n_err = popt[1], np.sqrt(pcov[1, 1])
            xf = np.logspace(*slider.value, 200)
            ax.loglog(xf, np.exp(popt[0]) * xf**n, '-', color='#D55E00', lw=2.5,
                      label=f'slope = {n:.3f} ± {n_err:.3f}')
            ax.axvspan(x_lo, x_hi, color='#009E73', alpha=0.10)
        ax.legend(fontsize=9, frameon=False)
        ax.set_xlabel('x');  ax.set_ylabel('y')
        plt.tight_layout();  plt.show()

slider.observe(draw, names='value')
display(widgets.VBox([slider, plot_out]))
draw()
```

---

### Multi-dataset sequential (Back / Save & next / Done)

Use this when fitting N datasets (generations, conditions, subjects) one at a time. Each step saves the range, seeds the next one from the previous, and accumulates results.

```python
import ipywidgets as widgets
from IPython.display import display, clear_output

# data_dict: {label: {'x': array, 'y': array}}
# fit_ranges: optional dict {label: (x_lo, x_hi)} for seeding

_saved   = {}           # {label: (x_lo, x_hi)} — filled as you navigate
_state   = {'idx': 0}
_container = widgets.VBox([])


def _refresh():
    labels = sorted(data_dict.keys())
    label  = labels[_state['idx']]
    x_all  = data_dict[label]['x']
    y_all  = data_dict[label]['y']
    valid  = (x_all > 0) & (y_all > 0) & np.isfinite(x_all) & np.isfinite(y_all)
    xv, yv = x_all[valid], y_all[valid]
    lo_max = float(np.log10(xv.max()))

    # Seed: use saved → use fit_ranges lookup → sensible default
    prev      = _saved.get(label, fit_ranges.get(label, (xv.min(), xv.max() * 0.4)))
    seed_lo   = float(np.clip(np.log10(max(prev[0], xv.min())), 0.0, lo_max))
    seed_hi   = float(np.clip(np.log10(max(prev[1], xv.min()*2)), 0.0, lo_max))

    slider   = widgets.FloatRangeSlider(
        value=[seed_lo, seed_hi], min=0.0, max=lo_max, step=0.05,
        description='log₁₀(x):', style={'description_width': '80px'},
        layout=widgets.Layout(width='600px'), readout_format='.2f',
    )
    plot_out = widgets.Output()
    info_lbl = widgets.HTML(value='')
    prog_lbl = widgets.HTML(
        value=f'<b>{label}</b>&nbsp;&nbsp;({_state["idx"]+1} / {len(labels)})'
    )
    b_back = widgets.Button(description='< Back',        button_style='',       layout=widgets.Layout(width='90px'))
    b_next = widgets.Button(description='Save & next >', button_style='success', layout=widgets.Layout(width='135px'))
    b_done = widgets.Button(description='Done',          button_style='primary', layout=widgets.Layout(width='90px'))

    def draw(change=None):
        lo, hi = slider.value
        x0, x1 = 10**lo, 10**hi
        mask    = (xv >= x0) & (xv <= x1)
        with plot_out:
            clear_output(wait=True)
            fig, ax = plt.subplots(figsize=(9, 4.5))
            ax.loglog(xv, yv, '.', ms=3, alpha=0.5, color=PALETTE[0])
            if mask.sum() >= 3:
                popt, pcov = curve_fit(linear_log, np.log(xv[mask]), np.log(yv[mask]))
                n, n_err = popt[1], np.sqrt(pcov[1, 1])
                xf = np.logspace(lo, hi, 200)
                ax.loglog(xf, np.exp(popt[0]) * xf**n, '-', color='#D55E00', lw=2.5,
                          label=f'slope={n:.3f}±{n_err:.3f}')
                ax.axvspan(x0, x1, color='#009E73', alpha=0.10)
                info_lbl.value = f'<b>slope = {n:.4f} ± {n_err:.4f}</b>'
            ax.legend(fontsize=9, frameon=False)
            ax.set_title(str(label))
            plt.tight_layout();  plt.show()

    def on_next(_):
        _saved[label] = (10**slider.value[0], 10**slider.value[1])
        if _state['idx'] < len(labels) - 1:
            _state['idx'] += 1
            _refresh()
        else:
            b_done.button_style = 'warning'

    def on_back(_):
        if _state['idx'] > 0:
            _state['idx'] -= 1
            _refresh()

    def on_done(_):
        _saved[label] = (10**slider.value[0], 10**slider.value[1])
        _container.children = [widgets.HTML('<b>Done. Ranges saved in _saved.</b>')]

    b_back.on_click(on_back)
    b_next.on_click(on_next)
    b_done.on_click(on_done)
    slider.observe(draw, names='value')

    _container.children = [
        prog_lbl,
        widgets.HBox([b_back, b_next, b_done]),
        slider, info_lbl, plot_out,
    ]
    draw()


display(_container)
_refresh()

# After clicking Done, run fits with saved ranges:
# fit_results = {}
# for label, (x_lo, x_hi) in _saved.items():
#     xv = data_dict[label]['x']; yv = data_dict[label]['y']
#     mask = (xv >= x_lo) & (xv <= x_hi) & (xv > 0) & (yv > 0)
#     popt, pcov = curve_fit(linear_log, np.log(xv[mask]), np.log(yv[mask]))
#     fit_results[label] = {'n': popt[1], 'n_err': np.sqrt(pcov[1,1]), 'A': np.exp(popt[0])}
```

---

### Auto-fit with interactive fallback

Try a default range first; only show the slider if residuals are too large. Avoids unnecessary interaction for well-behaved data.

```python
AUTO_LO, AUTO_HI = x_c[len(x_c)//5], x_c[4*len(x_c)//5]  # middle 60% as default

def fit_and_assess(x, y, x_lo, x_hi):
    mask = (x >= x_lo) & (x <= x_hi) & (x > 0) & (y > 0)
    if mask.sum() < 5:
        return None
    popt, pcov = curve_fit(linear_log, np.log(x[mask]), np.log(y[mask]))
    n, n_err = popt[1], np.sqrt(pcov[1, 1])
    A = np.exp(popt[0])
    residual = np.sqrt(np.mean(((y[mask] - A * x[mask]**n) / y[mask])**2))
    return {'n': n, 'n_err': n_err, 'A': A, 'residual': residual,
            'x_lo': x_lo, 'x_hi': x_hi}

result = fit_and_assess(x_c, y_c, AUTO_LO, AUTO_HI)

if result and result['residual'] < 0.05:          # < 5% residual: auto-fit is fine
    print(f"Auto-fit OK  slope={result['n']:.4f}±{result['n_err']:.4f}  "
          f"residual={result['residual']*100:.1f}%")
    # plot directly ...
else:
    print(f"Auto-fit residual {result['residual']*100:.1f}% > 5% — opening interactive slider")
    # paste single-dataset slider here ...
```

---

## Local Exponent / Running Slope

Visualizes how the power-law exponent evolves over the data range — useful for spotting crossovers and transients.

```python
# Subsample to ~500 log-spaced points first (gradient on dense data is very slow and noisy)
idx = np.unique(np.round(np.geomspace(1, len(x_c), 500)).astype(int)) - 1
log_x_sub = np.log(x_c[idx])
log_y_sub = np.log(y_c[idx])

alpha_local = np.gradient(log_y_sub, log_x_sub)

fig, ax = plt.subplots(figsize=(8, 5))
ax.semilogx(x_c[idx], alpha_local, '.', ms=3, alpha=0.7, color=PALETTE[0])
ax.axhline(expected_exponent, color='#7a7a7a', lw=1.5, ls='--',
           label=f'expected = {expected_exponent:.4f}')
ax.set_xlabel('x')
ax.set_ylabel(r'local exponent $d\ln y / d\ln x$')
ax.legend(fontsize=9, frameon=False)
```

**Edge artifact fix:** `np.gradient` spikes at array boundaries. Trim first and last ~2 points before plotting: `ax.semilogx(x_c[idx][2:-2], alpha_local[2:-2], ...)`.

---

## Scaling Collapse

When multiple datasets (indexed by a size parameter `L`) all follow the same scaling law, you can collapse them onto a single master curve by rescaling axes. This is a powerful visual confirmation of a scaling hypothesis.

**Idea:** if `y(x, L) = L^a · f(x / L^b)`, then plotting `y/L^a` vs `x/L^b` for all `L` should collapse to `f`.

```python
# Example: y ~ x^alpha for x << x_cross ~ L^b, y ~ const for x >> x_cross
# a = 2 (saturation scale), b = known or fitted exponent

a = 2.0    # saturation exponent (y_sat ~ L^a)
b = 2.32   # crossover exponent (x_cross ~ L^b) — use fitted or theoretical value

fig, ax = plt.subplots(figsize=(8, 6))
for i, (label, d) in enumerate(data_dict.items()):
    L = d['L']   # size parameter for this dataset
    xv = d['x'][d['x'] > 0]
    yv = d['y'][d['x'] > 0]
    ax.loglog(xv / L**b, yv / L**a,
              '-', color=PALETTE[i % len(PALETTE)], lw=1.8, alpha=0.8, label=label)

# Reference power law in the scaling regime
x_ref = np.logspace(-5, -0.5, 200)
ax.loglog(x_ref, 0.5 * x_ref**(1/b), '--', color='#7a7a7a', lw=1.5,
          label=f'$\\sim x^{{1/b}}$')

ax.set_xlabel(r'$x\, /\, L^b$')
ax.set_ylabel(r'$y\, /\, L^a$')
ax.legend(fontsize=9, frameon=False)
```

**Tip:** a clean collapse (curves on top of each other) confirms the exponents. A spread or systematic offset means the exponents need adjustment.

---

## Cumulative / Survival Distribution Fitting

For fitting power laws in probability distributions. Fitting the survival function `P(X > x) ~ x^(−α)` is more robust than fitting the PDF histogram (avoids binning artifacts).

```python
# Compute survival function from raw samples
data_sorted = np.sort(samples)[::-1]            # descending
survival    = np.arange(1, len(samples)+1) / len(samples)   # P(X > x)

# Filter to positive, finite values
valid = (data_sorted > 0) & np.isfinite(data_sorted)
x_surv = data_sorted[valid]
p_surv = survival[valid]

# Fit on a log-spaced grid (avoids high-density bias at small values)
x_grid = np.logspace(np.log10(x_surv.min()), np.log10(x_surv.max()), 200)
p_grid = np.interp(x_grid, x_surv[::-1], p_surv[::-1])

# Power-law fit: P(X > x) ~ x^(−α)  →  slope = −α
mask = (x_grid >= x_lo) & (x_grid <= x_hi) & (p_grid > 0)
popt, pcov = curve_fit(linear_log, np.log(x_grid[mask]), np.log(p_grid[mask]))
alpha = -popt[1];  alpha_err = np.sqrt(pcov[1, 1])

fig, ax = plt.subplots(figsize=(8, 5))
ax.loglog(x_surv, p_surv, '.', ms=2, alpha=0.4, color=PALETTE[0], label='Data')
xf = np.logspace(np.log10(x_lo), np.log10(x_hi), 200)
ax.loglog(xf, np.exp(popt[0]) * xf**popt[1], '-', color='#D55E00', lw=2.5,
          label=f'$P(X>x) \\propto x^{{-{alpha:.3f}}}$')
ax.axvspan(x_lo, x_hi, color='#009E73', alpha=0.10)
ax.legend(fontsize=9, frameon=False)
```

---

## Common Plot Problems

### Noisy scatter → add trend
```python
from scipy.ndimage import uniform_filter1d
order = np.argsort(x)
ax.scatter(x, y, s=4, alpha=0.4, rasterized=True)
ax.plot(x[order], uniform_filter1d(y[order], size=len(x)//30),
        color='#D55E00', lw=2)
```

### Overlapping legend
```python
ax.legend(loc='upper left', bbox_to_anchor=(1.01, 1), borderaxespad=0)
fig.tight_layout()
```

### Colorbar misaligned
```python
from mpl_toolkits.axes_grid1 import make_axes_locatable
cax = make_axes_locatable(ax).append_axes('right', size='4%', pad=0.08)
fig.colorbar(im, cax=cax, label='Intensity')
```

### Slow rendering / OOM
```python
ax.plot(x, y, rasterized=True)          # rasterize data, keep vector axes
ax.scatter(x, y, s=2, rasterized=True)
```

### Histogram bin count
```python
ax.hist(data, bins='auto')              # Sturges/auto
ax.hist(data, bins='fd')                # Freedman-Diaconis (heavy tails)
# Log-binned histogram for power-law data:
log_bins = np.logspace(np.log10(data.min()), np.log10(data.max()), 50)
counts, edges = np.histogram(data, bins=log_bins)
ax.loglog(edges[:-1]+np.diff(edges)/2, counts/np.diff(edges)/counts.sum(), 'o', ms=5)
```

---

## Saving Thesis Figures

```python
from pathlib import Path
FIG_DIR = Path('figures') / 'module_name'
FIG_DIR.mkdir(parents=True, exist_ok=True)

fig.patch.set_alpha(0.0)
ax.patch.set_alpha(0.0)
plt.tight_layout()
plt.savefig(FIG_DIR / 'descriptive_name.pdf',
            dpi=300, bbox_inches='tight', transparent=True)
```

Always: `dpi=300`, `bbox_inches='tight'`, `transparent=True`, PDF for print/thesis.

---

## Cross-Figure Checklist

After fixing one figure:
- Other notebooks with the same figure type? Apply the same fix.
- Every `savefig` in this notebook has `transparent=True` and `bbox_inches='tight'`?
- Same `PALETTE` and rcParams used throughout?
- LaTeX math in axis labels: `r'$\alpha$'`, `r'$\langle r^2 \rangle$'`?
