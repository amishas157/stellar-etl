# H005: restored-key parquet requests hit a nil-schema write path

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Parquet export request aborts restored-key batches
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes` supports `--export-restored-keys` together with `--write-parquet`, it should either emit a restored-key parquet file or reject that combination before processing the batch. A successful-looking run should not reach a parquet writer with no schema for a resource that already produced JSON rows.

## Mechanism

The command appends `transform.RestoredKeyOutput` rows into `transformedOutputs["restored_key"]`, but the parquet switch in `exportTransformedData()` has no `case transform.RestoredKeyOutput`. That leaves `skip == false`, `parquetSchema == nil`, and `transformedResource == nil`, so the post-loop `WriteParquet(transformedResource, parquetPath, parquetSchema)` call is reached with no schema for a resource that the command explicitly exported, which should abort the batch and drop the requested parquet output.

## Trigger

Run `export_ledger_entry_changes --export-restored-keys true --write-parquet` on a ledger range containing restored-key entries, such as the test range `58764192-58764193`.

## Target Code

- `cmd/export_ledger_entry_changes.go:112-126` — restored keys are transformed into `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:321-364` — the parquet switch lacks a `transform.RestoredKeyOutput` branch
- `cmd/export_ledger_entry_changes.go:370-371` — `WriteParquet(...)` is still called when `skip` remains false
- `internal/transform/schema.go:680-686` — `RestoredKeyOutput` exists as an exported JSON schema
- `internal/transform/parquet_converter.go:27-442` — there is no `RestoredKeyOutput.ToParquet()` implementation

## Evidence

The command test suite already covers restored-key JSON exports, so this is an active and reachable resource type. The absence of both a parquet schema branch and a `ToParquet()` implementation means the restored-key parquet path is structurally incomplete while still being reachable behind the global `write-parquet` flag.

## Anti-Evidence

This path likely fails loudly rather than silently corrupting values, so the severity is operational rather than structural. The precise runtime failure mode depends on how `parquet-go` reacts to a nil schema, but either way the user does not receive the requested restored-key parquet output.
