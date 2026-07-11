# How to Run This Pipeline

## 1. Prerequisites

You need Python 3.9+ and Jupyter installed. If you don't have Jupyter yet:

```bash
pip install notebook
```

(Or, if your group uses JupyterLab instead: `pip install jupyterlab`)

### Required Python packages

Install these before running either notebook:

```bash
pip install pandas pyarrow
```

- **pandas** — all data reading/cleaning/transformation.
- **pyarrow** — required for the `.parquet` outputs (`process.ipynb` reads
  and `transformation.ipynb`'s dimension/fact tables use Parquet under the
  hood via pandas). Without it, you'll get an
  `ImportError: Unable to find a usable engine` when saving/reading parquet files.

> **Note:** The first cell of `process.ipynb` runs
> `pip install pandas pyarrow pandera` from inside the notebook. On some
> systems (e.g. Debian/Ubuntu-based ones with "externally managed"
> environments) this fails with an `externally-managed-environment` error
> unless you add `--break-system-packages`, or unless you're using a virtual
> environment. **Safest approach: install the packages yourself in your
> terminal first (command above), then just skip/ignore that first cell** —
> everything else in the notebook will still run fine.

## 2. Folder structure this expects

Run the notebooks from inside the `finmark_pipeline/` folder. It expects:

```
finmark_pipeline/
├── data/
│   └── raw/
│       ├── event_logs.csv
│       ├── marketing_summary.csv
│       └── trend_report.csv
├── process.ipynb
└── transformation.ipynb
```

Everything else (`data/processed/`, `data/final_visuals/`,
`warehouse_outputs/`) is created automatically when you run the notebooks —
you don't need to make these folders yourself.

## 3. Run order (this matters)

Run the notebooks **in this order** — `transformation.ipynb` reads the
output of `process.ipynb`, so it will fail (or run on stale data) if you
run it first or skip `process.ipynb`.

1. **Open `process.ipynb`** in Jupyter → **Run All Cells**
   (Kernel menu → "Restart Kernel and Run All Cells" is the safest option,
   so you know it's running fresh and not reusing an old variable from a
   previous run).
   - Reads the 3 raw CSVs from `data/raw/`
   - Validates, deduplicates, masks `user_id`, and fills missing values
   - Writes cleaned CSVs to `data/processed/`
   - Writes `pipeline_run_report.json` — check this after running to
     confirm all 3 datasets show `"status": "success"`

2. **Open `transformation.ipynb`** → **Run All Cells** (same "Restart
   Kernel and Run All" approach).
   - Reads the cleaned files from `data/processed/`
   - Builds session tracking, dimension tables, fact tables
   - Writes warehouse outputs to `warehouse_outputs/`
   - Writes dashboard-ready CSVs to `data/final_visuals/`

## 4. How to tell if it worked

After both notebooks finish:

- `data/processed/pipeline_run_report.json` should show `"status": "success"`
  for all three datasets (`event_logs`, `marketing_summary`, `trend_report`).
  If any show `"status": "failed"`, the error message is right there in the
  same entry — that dataset's raw CSV likely has a real problem (missing
  file, corrupted encoding, totally different structure), not a bug in the
  notebook.
- `warehouse_outputs/` should contain `dim_users.csv`, `dim_products.csv`,
  `dim_event_types.csv`, `dim_dates.csv`, `fact_events.csv`,
  `fact_sessions.csv`, `fact_weekly_trends.csv`, `fact_daily_marketing.csv`,
  and three `forecast_*.csv` files.
- `data/final_visuals/` should contain `dashboard_event_type.csv`,
  `dashboard_daily_kpi.csv`, `dashboard_session_summary.csv`.
- Open `warehouse_outputs/dim_users.csv` and confirm `user_id` values look
  like random 16-character hex strings (e.g. `02445e3b05611b85`) — **not**
  like `U0001`. If you ever see plain `U####`-style IDs anywhere outside
  `data/raw/`, something upstream isn't masking correctly.

## 5. If something goes wrong

- **`ImportError: Unable to find a usable engine; tried using: 'pyarrow'`**
  → run `pip install pyarrow` in your terminal, then restart the kernel.
- **`FileNotFoundError` on one of the raw CSVs** → confirm you're running
  the notebook from inside `finmark_pipeline/` (not a parent folder), and
  that `data/raw/` has all three CSVs.
- **One dataset shows `"status": "failed"` in the run report** → this is
  expected, resilient behavior, not a crash — read the `"error"` message in
  that dataset's entry to see what was wrong with that specific file. The
  other datasets will have processed normally.
- **`NameError: name 'Path' is not defined`** → this means you're running
  an older copy of `process.ipynb` that's missing an import. Make sure
  you're using the version from this zip.
