# industrial-demand-response-study

Cost-optimal production scheduling for a two-machine industrial process (Chiron 800 CNC mill,
HML MP 50 1 VK chip press) against German day-ahead electricity prices. A Pyomo MILP shifts
production batches into cheap 15-minute price slots — subject to inventory, storage and pickup
deadline constraints — and is benchmarked against a price-blind ASAP baseline over all 252
business days of 2024.

**Headline result:** 2,423.94 € optimized vs. 2,948.49 € ASAP → **524.55 € (17.79 %) annual
electricity cost saving**, at identical production output (1,687 press runs, 675 mill runs in both
scenarios).

## Setup

Python 3.13 with a local `.venv`:

```bash
python -m venv .venv && .venv/Scripts/activate && pip install pandas numpy pyomo highspy plotly kaleido matplotlib jupyter
```

The MILP is solved with HiGHS through Pyomo's `appsi_highs` interface (`highspy`, no external
solver binary needed).

## Structure

| Path | Contents |
| --- | --- |
| `optimization_model.ipynb` | The model: parameters, order book, single-day smoke test, rolling-horizon full-year run, validation |
| `paper_figures.ipynb` | Publication figures (B, C, F, G, H) built from persisted results, no re-solve |
| `data/` | Day-ahead prices (15-min, 2018–2024), synthetic order book, business-day calendar, CO₂ intensity |
| `templates/` | 1-min machine load templates for one mill / press session, plus the template assessor notebook |
| `results/` | Run outputs (see below) |
| `figures/` | Exported figures, `figures/paper/` holds the 300 dpi paper set |
| `utils/` | Helper notebooks (business-day calendar, CO₂ intensity fetch) |

The two raw 5-second machine recordings in `data/` are gitignored (>100 MB); the model only needs
the small aggregated templates in `templates/`.

## Running it

Run `optimization_model.ipynb` top to bottom — it is a linear script, later cells depend on state
from earlier ones. Two stages produce output:

1. **Single-day smoke test** (`MODEL_DAY`, default `2024-03-18`) →
   `results/run_<day>_8-22_<label>/` with `schedule_optimized.csv`, `schedule_asap.csv`,
   `parameters.json` (full reproducible config) and five PNG plots.
2. **Rolling two-day horizon, full year 2024** → `results/year_2024/` with `daily_results.csv`
   (one row per business day), `schedule_opt_long.csv` / `schedule_asap_long.csv` (15-min
   resolution) and `run_config.json`.

The year loop is cached: if `results/year_2024/daily_results.csv` exists it is not re-solved. Set
`FORCE_RERUN = True` to solve anyway. A full re-solve takes roughly 10–15 minutes (252 window
MILPs, all terminating optimal). Eight validation checks read the persisted CSVs back and assert
conservation of housings and modules, non-binding capacity caps, day-boundary integrity and energy
consistency.

Then run `paper_figures.ipynb`, which reads `results/year_2024/` and writes to `figures/paper/`.

Results are deterministic: fixed seed (187) for the synthetic order book, HiGHS at `mip_gap = 0.0`.
Re-running reproduces every number bit-for-bit apart from solver timings.
