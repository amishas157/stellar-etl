# H006: `auto_bump_ledgers` looks omitted from config-setting exports but has no live XDR source

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Config-setting completeness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the active `STATE_ARCHIVAL_SETTINGS` XDR arm still exposed an `auto_bump_ledgers` field, `TransformConfigSetting()` should carry that value into both JSON and Parquet exports instead of leaving `auto_bump_ledgers` at its zero default.

## Mechanism

The local schema, Parquet schema, and tests still define `AutoBumpLedgers`, but `TransformConfigSetting()` never reads or assigns any source value for that column. At first glance that looks like silent data loss for state-archival config rows.

## Trigger

Inspect `config_settings` export code for state-archival entries and compare the output struct fields against the transformer assignments.

## Target Code

- `internal/transform/schema.go:610` — JSON schema still exposes `auto_bump_ledgers`
- `internal/transform/schema_parquet.go:352` — Parquet schema still exposes `auto_bump_ledgers`
- `internal/transform/parquet_converter.go:393` — Parquet conversion forwards the field if it were ever populated
- `internal/transform/config_setting.go:85-107,159-169` — state-archival settings are read, but no `AutoBumpLedgers` source is assigned
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/xdr/xdr_generated.go:60818-60829` — current `StateArchivalSettings` has no `AutoBumpLedgers` member

## Evidence

The ETL's output structs still carry `AutoBumpLedgers`, and the transformer's state-archival block reads neighboring fields like `MaxEntryTtl`, `MinTemporaryTtl`, `MaxEntriesToArchive`, and `LiveSorobanStateSizeWindowSampleSize` but never populates the auto-bump column. The same stale field also survives in the upstream `github.com/stellar/go/processors/config_setting` output struct, where it likewise remains unassigned.

## Anti-Evidence

The current generated XDR for `StateArchivalSettings` no longer contains an `AutoBumpLedgers` field at all, so there is no live on-chain value for this repository to drop. This appears to be dead compatibility schema rather than an active export corruption path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no current XDR source field to map. `auto_bump_ledgers` is a stale output column that remains zero because the protocol no longer exposes a corresponding `StateArchivalSettings` member, so the transform is not losing live data.

### Lesson Learned

When a config-setting output column is present in schema/tests but absent from current generated XDR, treat it as a dead compatibility artifact first, not as proof of a dropped export field.
