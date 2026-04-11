# H004: Restored-key JSON rows are dropped into an empty parquet file

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes --export-restored-keys --write-parquet` produces restored-key JSON rows for a batch, the corresponding parquet file should contain the same restored-key rows.

## Mechanism

The restored-key path appends `transform.RestoredKeyOutput` rows into `transformedOutputs["restored_key"]`, but the downstream Parquet type switch has no `RestoredKeyOutput` case. As a result, the JSON file is populated while `transformedResource` stays empty and `WriteParquet()` emits a zero-row parquet file, silently erasing every restored key from the Parquet side of the export.

## Trigger

Run `stellar-etl export_ledger_entry_changes --export-restored-keys true --write-parquet ...` over any batch that contains restored ledger entries, such as the repository's existing restored-key test range `58764192-58764193`.

## Target Code

- `cmd/export_ledger_entry_changes.go:97-100` — maps `--export-restored-keys` to the `restored_key` output bucket
- `cmd/export_ledger_entry_changes.go:112-125` — transforms restored ledger changes into `transform.RestoredKeyOutput` rows
- `cmd/export_ledger_entry_changes.go:321-364` — Parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — still calls `WriteParquet()` after the unsupported rows were never appended
- `internal/transform/restored_key.go:40-46` — defines the JSON row that currently has no Parquet handling
- `cmd/export_ledger_entry_changes_test.go:97-101` — demonstrates that restored-key JSON export is a live, covered path

## Evidence

The export mapping and transformation loop clearly populate restored-key JSON output today, but the Parquet switch never mentions `RestoredKeyOutput`. I also confirmed that the parquet library accepts `WriteParquet(nil, path, nil)` for zero rows, which turns this into silent data loss rather than an obvious crash.

## Anti-Evidence

This is Parquet-only: the JSON batch remains correct, and users who do not enable `--write-parquet` will not see the issue. The actual on-disk behavior depends on the parquet library tolerating the empty write, but that tolerance is exactly what makes the drop silent.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from `export_ledger_entry_changes` flag parsing through restored-key transformation to Parquet output. Confirmed that `RestoredKeyOutput` has no `ToParquet()` method, no `RestoredKeyOutputParquet` struct exists in `schema_parquet.go`, and no `case transform.RestoredKeyOutput` exists in the type switch at lines 322-364 of `export_ledger_entry_changes.go`. The `skip` variable defaults to `false` (line 314), so `WriteParquet(nil, parquetPath, nil)` is unconditionally called for the `restored_key` resource when `--write-parquet` is enabled. This contrasts with `ClaimableBalanceOutput` which has an explicit `skip = true` (line 335) and a comment explaining the intentional omission.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:100` — `"export-restored-keys": {"restored_key"}` maps flag to output bucket
- `cmd/export_ledger_entry_changes.go:103-109` — initializes `transformedOutputs["restored_key"] = []interface{}{}` when flag is set
- `cmd/export_ledger_entry_changes.go:111-127` — transforms restored changes into `RestoredKeyOutput` and appends to `transformedOutputs["restored_key"]`
- `cmd/export_ledger_entry_changes.go:312-314` — `var transformedResource []transform.SchemaParquet`, `var parquetSchema interface{}`, `var skip bool` all zero-valued per resource iteration
- `cmd/export_ledger_entry_changes.go:322-364` — type switch has 10 cases (Account, AccountSigner, ClaimableBalance, ConfigSetting, ContractCode, ContractData, Pool, Offer, Trustline, Ttl) but NO case for `RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — `!skip && writeParquet` evaluates true, calls `WriteParquet(nil, parquetPath, nil)`
- `cmd/command_utils.go:162-180` — `WriteParquet` creates file, creates writer with nil schema, iterates nil data slice
- `internal/transform/schema.go:679-686` — `RestoredKeyOutput` struct (5 flat fields, no nested types)
- `internal/transform/parquet_converter.go:15-17` — `SchemaParquet` interface requires `ToParquet()` — `RestoredKeyOutput` does NOT implement this
- `internal/transform/schema_parquet.go` — no `RestoredKeyOutputParquet` struct exists (all other non-skipped types have one)

### Findings

1. **Missing type switch case**: The Parquet type switch in `exportTransformedData()` has no `case transform.RestoredKeyOutput`. All other exported entity types either have a case (10 types) or an explicit `skip = true` (ClaimableBalance). RestoredKey has neither.

2. **No Parquet schema or converter**: Unlike every other entity type, `RestoredKeyOutput` has no `RestoredKeyOutputParquet` struct in `schema_parquet.go` and no `ToParquet()` method in `parquet_converter.go`.

3. **No intentional skip**: Unlike `ClaimableBalanceOutput` which has an explicit `skip = true` with a comment explaining the omission, `RestoredKeyOutput` simply falls through the switch silently. The `skip` variable defaults to `false`.

4. **Triggers on every batch**: Even batches with zero restored keys trigger this path because `transformedOutputs["restored_key"]` is always initialized (line 106), causing the outer loop to iterate over it and call `WriteParquet(nil, parquetPath, nil)`.

5. **Not intentional**: The `RestoredKeyOutput` struct has 5 simple flat fields (string, string, uint32, time.Time, uint32) — there is no technical reason to skip Parquet conversion (unlike `ClaimableBalance` which has nested recursive XDR types). This is an omission, not a design decision.

6. **Runtime behavior**: With nil schema, `writer.NewParquetWriter` likely either panics or returns an error triggering `cmdLogger.Fatal`, crashing the entire batch export. If the parquet library tolerates nil schema, it produces a zero-row file (silent data loss). Either outcome is a bug.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` — extend existing test infrastructure
- **Setup**: Use the existing restored-key test range `58764192-58764193` with `--write-parquet` enabled
- **Steps**: 
  1. Run `export_ledger_entry_changes` with `--export-restored-keys true --write-parquet` flags
  2. Verify JSON output contains restored key rows (existing test already does this)
  3. Check the corresponding parquet file
- **Assertion**: Either (a) the parquet file should contain the same number of rows as JSON output, or (b) if the write crashes, that crash confirms the bug. The PoC should verify that the type switch at line 322 is the root cause by checking that `RestoredKeyOutput` rows are present in `output` but absent from `transformedResource` after the loop.
