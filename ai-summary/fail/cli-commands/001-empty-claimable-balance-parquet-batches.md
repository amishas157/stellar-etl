# H001: Empty claimable-balance batches still emit schema-less parquet files

**Date**: 2026-04-15
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If claimable-balance parquet export is intentionally unsupported, `export_ledger_entry_changes --export-balances --write-parquet` should consistently emit no `claimable_balances` parquet artifact, including batches that happen to contain zero claimable-balance rows.

## Mechanism

`exportTransformedData` only flips `skip = true` when it actually iterates over a `transform.ClaimableBalanceOutput`. For an empty `claimable_balances` batch, the loop never runs, `skip` stays false, and the function falls through to `WriteParquet(nil, parquetPath, nil)`, which can create a schema-less empty parquet file even though the surrounding code comment says this resource is intentionally skipped.

## Trigger

Run `export_ledger_entry_changes --export-balances --write-parquet` across a batch whose change set includes no claimable-balance entries.

## Target Code

- `cmd/export_ledger_entry_changes.go:exportTransformedData:304-372` — initializes `skip` false, only sets it inside the row loop, then calls `WriteParquet(...)` when `!skip`

## Evidence

The only `ClaimableBalanceOutput` branch sets `skip = true` inside the per-row switch. An empty batch reaches `if !skip && writeParquet { WriteParquet(...) }` with `skip == false` and `parquetSchema == nil`.

## Anti-Evidence

This only affects zero-row batches: no incorrect claimable-balance row is ever written, and earlier investigations already established that parquet-go handles `nil` schema plus zero rows without crashing.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The anomaly creates at most a zero-row, schema-less placeholder file for an already unsupported parquet resource. It does not silently change any exported claimable-balance value, and downstream consumers are more likely to reject or visibly special-case the file than to ingest wrong-but-plausible data from it.

### Lesson Learned

For this objective, an unsupported-format inconsistency is not enough by itself. Prefer hypotheses that produce incorrect row values or row presence in otherwise normal-looking exports, not ones that only create obviously odd empty artifacts.
