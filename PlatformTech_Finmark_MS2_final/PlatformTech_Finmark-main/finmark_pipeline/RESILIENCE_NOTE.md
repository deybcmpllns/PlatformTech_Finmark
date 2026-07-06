# Resilience Note

This note describes how the pipeline detects missing or broken columns before
processing, and how it keeps running when the source data isn't perfect.

## 1. Schema is declared first, never inferred

`process.ipynb` declares the expected columns and types up front
(`IMPORTANT_COLUMNS`, `EXPECTED_DTYPES`) before any file is read. Every raw
file is checked against this declared schema rather than trusting whatever
columns happen to survive the read.

## 2. Missing columns are detected and filled

If a required column is completely absent from a source file,
`add_missing_columns()` adds it with a hard fallback value (`0.0`, `"unknown"`,
or `1970-01-01`, depending on type) and prints which column triggered it. This
is the one case where a fixed constant is appropriate — there's no real data
to learn a distribution from.

## 3. Missing *values* are filled from the data's own distribution, not a fixed constant

For columns that exist but have missing values, `eda_impute_column()` fills
them using the column's own distribution instead of a blanket constant:
- Numeric columns → median (per-`event_type` median for `amount`, since an
  EDA pass showed `amount` is missing ~51% of the time evenly across every
  event type, not just non-transactional ones — a flat `0.0` would have cut
  the true mean roughly in half).
- String columns → mode (most frequent value).
- A hard constant is used only if a column has zero real values anywhere to
  learn from.

## 4. Row-level validation and deduplication (L0)

Before any schema or PII work, `validate_and_deduplicate()` drops rows
missing a required key column (`user_id`/`event_type`/`event_time` for
events, `date` for marketing, `week` for trends) and removes exact duplicate
rows. This runs once, in `process.ipynb`, so bad rows never reach later
layers.

## 5. PII masking (L1)

`mask_pii_columns()` replaces `user_id` with a salted SHA-256 hash right
after L0. The hash is deterministic, so joins and groupbys on `user_id`
still work the same downstream — but the real ID is never exposed past this
point.

## 6. One bad file doesn't take down the whole run

Each dataset is processed inside a `try/except` in the main loop. If a file
is corrupted or unreadable (bad encoding, unparseable structure), that
dataset is logged as `"failed"` with the specific error message in
`pipeline_run_report.json`, and the loop continues to the next dataset
instead of crashing the notebook.

**Tested**: intentionally corrupted `trend_report.csv` with invalid encoding.
Result — `event_logs` and `marketing_summary` still processed successfully;
`trend_report` was cleanly logged as failed with its error message, and the
notebook ran to completion.

## What this doesn't cover yet

- Session tracking (`transformation.ipynb`) still uses a 30-minute gap
  threshold; given how sparse the event data is (~5 events/user spread over
  days), a 24-hour threshold is more appropriate and is planned as a
  follow-up.
- Schema mismatches that produce a *parseable but entirely wrong* file
  (e.g. a CSV with none of the expected columns) currently fall back to a
  single all-fallback row rather than being flagged as a critical schema
  failure. Worth a stricter check in a future pass.
