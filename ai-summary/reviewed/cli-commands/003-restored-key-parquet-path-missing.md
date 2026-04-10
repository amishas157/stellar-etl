# H003: Restored-key ledger-entry exports have no viable parquet path

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` runs with `--export-restored-keys` and `--write-parquet`, the command should emit a parquet file whose restored-key rows match the JSON `restored_key` export for the same batch. The batch should not stop after producing only one of the two formats.

## Mechanism

The restored-key transform exists for JSON output, but `exportTransformedData` has no parquet switch case for `transform.RestoredKeyOutput`, and `internal/transform/schema_parquet.go` defines no `RestoredKeyOutputParquet`. That leaves `transformedResource` empty and `parquetSchema` nil for the `restored_key` resource, so the function reaches `WriteParquet(...)` without a schema capable of serializing the current batch's restored-key rows.

## Trigger

1. Run `export_ledger_entry_changes` with `--export-restored-keys` and `--write-parquet`.
2. Feed it any ledger range containing `LedgerEntryRestored` changes.
3. The JSON `restored_key` file is populated, but the parquet path for that same resource cannot be written correctly and the batch can terminate after partial output.

## Target Code

- `cmd/export_ledger_entry_changes.go:112-126` — appends `transform.RestoredKeyOutput` values into `transformedOutputs["restored_key"]`
- `cmd/export_ledger_entry_changes.go:321-364` — parquet type switch omits `transform.RestoredKeyOutput`
- `cmd/export_ledger_entry_changes.go:370-372` — still calls `WriteParquet(transformedResource, parquetPath, parquetSchema)`
- `internal/transform/schema.go:679-686` — JSON-side `RestoredKeyOutput` exists
- `internal/transform/schema_parquet.go:300-399` — no `RestoredKeyOutputParquet` exists alongside adjacent parquet schemas

## Evidence

The CLI explicitly exposes `export-restored-keys` and routes restored entries through `TransformRestoredKey`, so the JSON half of the feature is implemented. The parquet half has no matching schema type and no switch arm, unlike every other supported ledger-entry-change resource except the explicitly skipped claimable-balance case.

## Anti-Evidence

This issue is limited to parquet mode. JSON output for restored keys appears intentionally wired and should still contain correct rows when `--write-parquet` is not requested.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full execution path from CLI flag parsing through `exportTransformedData`. When `writeParquet=true` and the resource is `"restored_key"`, the type switch at lines 322-364 has no case for `transform.RestoredKeyOutput`. Because Go's zero value for `bool` is `false`, the `skip` variable stays `false`, `transformedResource` stays nil, and `parquetSchema` stays nil. The guard at line 370 (`!skip && writeParquet`) evaluates to `true`, so `WriteParquet(nil, parquetPath, nil)` is called. Inside `WriteParquet`, `writer.NewParquetWriter(parquetFile, nil, 1)` receives a nil schema, which will cause a fatal error or panic, killing the entire process.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:100` — `"export-restored-keys"` maps to `{"restored_key"}`, creating the key in `transformedOutputs`
- `cmd/export_ledger_entry_changes.go:112-126` — populates `transformedOutputs["restored_key"]` with `RestoredKeyOutput` values via `TransformRestoredKey`
- `cmd/export_ledger_entry_changes.go:274-289` — passes `transformedOutputs` (including `"restored_key"`) to `exportTransformedData`
- `cmd/export_ledger_entry_changes.go:304` — `for resource, output := range transformedOutput` iterates over all resources including `"restored_key"`
- `cmd/export_ledger_entry_changes.go:312-314` — `var transformedResource`, `parquetSchema`, `skip` all initialized to zero values (nil, nil, false)
- `cmd/export_ledger_entry_changes.go:321-364` — type switch has NO case for `transform.RestoredKeyOutput`; switch falls through without executing any case
- `cmd/export_ledger_entry_changes.go:370-372` — `!skip && writeParquet` is `true`, calls `WriteParquet(nil, parquetPath, nil)`
- `cmd/command_utils.go:162-180` — `WriteParquet` calls `writer.NewParquetWriter(parquetFile, nil, 1)` with nil schema, triggering fatal/panic
- `internal/transform/schema.go:679-686` — `RestoredKeyOutput` struct exists for JSON output
- `internal/transform/schema_parquet.go` — confirmed NO `RestoredKeyOutputParquet` type exists
- `internal/transform/parquet_converter.go:15-17` — `SchemaParquet` interface requires `ToParquet()`; `RestoredKeyOutput` does not implement it

### Findings

The bug is confirmed. When `--export-restored-keys` and `--write-parquet` are both active and the batch contains restored ledger entries:

1. **No parquet schema exists**: `RestoredKeyOutputParquet` is not defined anywhere in the codebase.
2. **No `ToParquet()` method**: `RestoredKeyOutput` does not implement the `SchemaParquet` interface.
3. **No switch case**: The type switch in `exportTransformedData` has no arm for `RestoredKeyOutput`.
4. **No skip guard**: Unlike `ClaimableBalanceOutput` (which explicitly sets `skip = true`), there is no handling at all for `RestoredKeyOutput`.
5. **Fatal crash**: `WriteParquet` is called with nil schema, causing `cmdLogger.Fatal` which kills the entire process.
6. **Non-deterministic blast radius**: Since Go map iteration is randomized, if `"restored_key"` is processed before other resources, ALL resource types in that batch lose their data.

This is a Pattern 3 (export command consistency) finding: every other resource type either has a complete parquet path (switch case + schema + `ToParquet()`) or an explicit `skip = true` guard. `restored_key` has neither.

### PoC Guidance

- **Test file**: `cmd/export_ledger_entry_changes_test.go` (or create a focused unit test for `exportTransformedData`)
- **Setup**: Create a `transformedOutputs` map with a single `"restored_key"` entry containing one `RestoredKeyOutput` value. Set `writeParquet = true`. Use a temp directory for output paths.
- **Steps**: Call `exportTransformedData(0, 1, tmpDir, tmpParquetDir, transformedOutputs, "", "", "", nil, true)`
- **Assertion**: Expect a panic or fatal error from `WriteParquet` when it receives nil schema. The test should recover from the panic and assert it occurred with a message related to nil schema or parquet writer initialization.
