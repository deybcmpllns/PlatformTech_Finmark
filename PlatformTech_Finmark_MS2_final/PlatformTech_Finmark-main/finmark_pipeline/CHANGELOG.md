# Changelog — FinMark Pipeline Resilience Pass

This documents every change made to `process.ipynb` and `transformation.ipynb`
during this round of fixes, in the order they were made.

---

## 1. L2 — Fixed the fallback logic to actually match the EDA

**Problem:** `process.ipynb` assumed `amount` was a "structural zero" for any
non-`checkout` event (`STRUCTURAL_ZERO_COLUMNS`) and force-filled those rows
with `0.0`. An EDA pass showed this was wrong — `amount` is genuinely missing
~51% of the time, spread evenly across *every* event type, not just
non-checkout ones. The rule was silently zeroing out ~874 real missing
values instead of estimating them.

**Fix:**
- Removed `"amount": ("event_type", "checkout")` from `STRUCTURAL_ZERO_COLUMNS`.
- Changed the imputation group key in `GROUP_IMPUTE_COLUMNS` from
  `product_id` to `event_type`, matching the EDA finding.

**Verified:** cleaned `amount` mean went from **809.29 → 1,612.59**
(raw non-null mean is ~1,598.99), and zero-forced rows went from 926 → 0.

---

## 2. L0 — Consolidated validation & deduplication

**Problem:** Row-level validation (dropping rows with missing key columns)
was split across two notebooks — `process.ipynb` only did a full-row
`drop_duplicates()`, while the actual null-key check happened later in
`transformation.ipynb`, *after* schema standardization had already run.

**Fix:**
- Added `NULL_KEY_COLUMNS` config and a new `validate_and_deduplicate()`
  function in `process.ipynb`. It drops rows missing a required key column
  (`user_id`/`event_type`/`event_time` for events, `date` for marketing,
  `week` for trends) and removes exact duplicates — now the **first** step
  after reading and standardizing column names, before any schema or PII
  work.
- Removed the now-redundant `dropna()`/`drop_duplicates()` calls from
  `transformation.ipynb`, keeping only the dtype re-coercion there (still
  needed since CSVs lose dtype info on read).

**Verified:** 0 nulls remain in key columns after cleaning; 0 duplicate rows.

---

## 3. L1 — Added PII masking (this layer didn't exist before)

**Problem:** `user_id` flowed through every layer — processed files,
warehouse dimension tables, dashboard exports — in plain text. Confirmed via
direct grep (`hash`, `mask`, `sha256`, `salt`, `pii`, etc. — zero matches)
and by comparing `user_id` values in raw vs. warehouse output (identical).

**Fix:**
- Added `PII_SALT` and `PII_COLUMNS` config, and a new `mask_pii_columns()`
  function in `process.ipynb`. Runs right after L0, before schema work.
  Replaces `user_id` with a salted SHA-256 hash (16 hex chars) — same input
  always produces the same output, so joins/groupbys on `user_id` still work
  identically downstream, but the real ID is never exposed past this point.

**Verified:** `warehouse_outputs/dim_users.csv` now shows hashed IDs
(e.g. `02445e3b05611b85`) instead of raw ones (`U0001`); `fact_events` and
`fact_sessions` have zero null `user_key`s, confirming joins still resolve.

---

## 4. L1 — Closed a PII leak caused by a leftover buggy cell

**Problem:** After the L1 fix above, `data/staging/dim_users.csv` still
showed **raw, unmasked** `user_id` values. Root cause: a buggy duplicate
cell in `transformation.ipynb` (previously flagged in milestone feedback as
clutter to delete) had its own loader that preferred `.parquet` over `.csv`.
Since `process.ipynb` only ever wrote `.csv`, a stale `.parquet` file left
over from an earlier run — containing raw `user_id` and a fabricated schema
(`event_id`, `transaction_amount`, `timestamp`, `session_id`) — was silently
picked up and re-exported to `data/staging/`.

**Fix:**
- Deleted the buggy duplicate cell from `transformation.ipynb` entirely.
  Everything it produced already existed correctly (and was already
  produced from masked data) via the legitimate cells earlier in the same
  notebook, exported to `warehouse_outputs/`.
- Deleted the stale `.parquet` files under `data/processed/` and the
  `data/staging/` folder so nothing unmasked was left sitting around.

**Verified:** `data/staging/` no longer exists; grepped every output
directory for raw `user_id` patterns — the only match is the original
`data/raw/event_logs.csv`, which is expected.

---

## 5. Mission requirement — per-dataset error isolation

**Problem:** The pipeline had no resilience against a genuinely broken
input file. Tested directly: corrupting `trend_report.csv` with invalid
byte encoding caused an uncaught `UnicodeDecodeError` that crashed the
entire notebook — `event_logs` and `marketing_summary` never got processed
either, and `data/processed/` was left holding **stale output from a
previous run with no warning that anything had failed**.

**Fix:** Wrapped the per-dataset processing loop in `process.ipynb` in a
`try/except`. On failure, the dataset is logged with `"status": "failed"`
and the specific error message in `pipeline_run_report.json`, and the loop
continues to the next dataset instead of crashing. Minimal diff — same
variable names, same summary-dict shape, only the loop body changed
(`process_dataset()` itself was not touched).

**Verified:** re-ran the exact same corrupted-file test —
`event_logs` and `marketing_summary` processed successfully, `trend_report`
was cleanly logged as `"failed"` with its error message, and the notebook
ran to completion. Re-ran the clean/uncorrupted data afterward to confirm
no regression (`amount` mean still ~1,612.59, all 3 datasets `"success"`).

---

## 6. Added `RESILIENCE_NOTE.md`

New file documenting all of the above — schema-first validation, EDA-based
fallback, L0 dedup, L1 masking, and per-dataset error isolation — plus an
honest "what this doesn't cover yet" section.

---

## Still open (not fixed this round)

- **L3 session tracking** uses a 30-minute gap threshold. Given how sparse
  the event data is (~5 events/user spread over days), a 24-hour threshold
  was the agreed target and is still outstanding.
- **Schema-mismatch detection**: a file that parses but has *none* of the
  expected columns currently falls through to an all-fallback row rather
  than being flagged as a critical schema failure. Noted in
  `RESILIENCE_NOTE.md` as a future improvement.
