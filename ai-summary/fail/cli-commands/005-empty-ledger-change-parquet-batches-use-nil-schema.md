# H004: Empty ledger-entry-change batches still enter parquet generation with a nil schema

**Date**: 2026-04-10
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a selected ledger-entry resource has zero rows in a batch, `export_ledger_entry_changes` should either skip parquet creation for that batch or create a valid empty parquet file using the resource's schema. An empty batch should not break the whole export.

## Mechanism

`transformedOutputs` is pre-populated with empty slices for every selected resource before the batch is examined. In `exportTransformedData`, the parquet schema is only assigned inside the `for _, o := range output` loop. When `output` is empty, `skip` stays false, `parquetSchema` stays nil, and the function still calls `WriteParquet(...)`; the parquet writer code dereferences its schema handler during finalization, so an empty-but-selected resource can stop the export instead of yielding an empty result set.

## Trigger

Run `export_ledger_entry_changes` with `--write-parquet=true` for a selected resource whose batch has no rows, such as `--export-config-settings=true` on ledgers `59834400-59834415` (the same empty-range case already covered by JSON-only tests). The JSON path creates an empty file, but the parquet path reaches `WriteParquet` without ever choosing a schema.

## Target Code

- `cmd/export_ledger_entry_changes.go:103-109` — selected resources are initialized even when the batch contains no matching changes
- `cmd/export_ledger_entry_changes.go:304-364` — schema selection happens only inside the per-row loop
- `cmd/export_ledger_entry_changes.go:370-372` — parquet write still runs with `skip == false`
- `cmd/command_utils.go:162-180` — `WriteParquet` does not guard against a nil schema before creating the writer

## Evidence

There is already a JSON-only regression test proving empty config-setting batches are expected and valid output (`cmd/export_ledger_entry_changes_test.go:138-144`). In the current implementation, the parquet schema is inferred from the first row instead of from the resource type, so empty output slices have no way to initialize `parquetSchema` before `WriteParquet` is invoked.

## Anti-Evidence

If at least one row of a supported type appears in the batch, the switch sets `parquetSchema` and the parquet path can proceed normally. This only manifests on empty batches for selected resource types.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (distinct from success/003 which covers missing switch case for non-empty data; this covers empty batches)
**Failed At**: reviewer

### Trace Summary

The hypothesis correctly identifies that empty batches reach `WriteParquet` with nil schema and nil data. However, the claimed crash mechanism is wrong. Tracing through the parquet-go library (v1.6.2) confirms that `NewParquetWriter(file, nil, 1)` succeeds — the nil schema check at `writer.go:83` simply skips schema setup. With no rows written, `flushObjs()` returns immediately at line 239 (empty Objs slice), `Flush()` skips the row-group building block at line 349 (empty PagesMapBuf), and `RenameSchema()` iterates over empty `Footer.Schema` and `Footer.RowGroups` slices without ever dereferencing the nil `SchemaHandler`. The result is a valid empty parquet file with PAR1 header/footer and no columns.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:88-109` — confirmed empty slices are pre-populated for all selected resources
- `cmd/export_ledger_entry_changes.go:304-377` — confirmed that when `output` is empty, the `for _, o := range output` loop body never executes, leaving `skip=false`, `parquetSchema=nil`, `transformedResource=nil`
- `cmd/export_ledger_entry_changes.go:370-371` — confirmed `WriteParquet(nil, parquetPath, nil)` is called
- `cmd/command_utils.go:162-180` — `WriteParquet` passes nil schema to `writer.NewParquetWriter`
- `parquet-go@v1.6.2/writer/writer.go:56-100` — `NewParquetWriter` with nil obj: line 83 `if obj != nil` skips all schema setup, returns successfully with nil `SchemaHandler`
- `parquet-go@v1.6.2/writer/writer.go:236-241` — `flushObjs` returns nil immediately when `len(pw.Objs) <= 0`
- `parquet-go@v1.6.2/writer/writer.go:342-349` — `Flush` skips row-group building when `len(pw.PagesMapBuf) == 0`
- `parquet-go@v1.6.2/writer/writer.go:114-126` — `RenameSchema` iterates over empty `Footer.Schema` (no iterations = no nil SchemaHandler dereference)
- `parquet-go@v1.6.2/writer/writer.go:129-203` — `WriteStop` writes empty footer, footer size, and "PAR1" trailer successfully

### Why It Failed

The claimed mechanism — that the parquet writer dereferences its nil schema handler during finalization, crashing the export — is incorrect. The parquet-go library handles nil schema gracefully when no rows are written: `NewParquetWriter` skips schema setup for nil obj, `Flush` skips row-group construction when PagesMapBuf is empty, and `RenameSchema` loops over empty collections without touching the nil SchemaHandler. The result is a valid empty parquet file (PAR1 magic + empty footer + PAR1 magic), which is consistent with the JSON path that also produces an empty file for the same case. The export is not stopped. This is also corroborated by the success/003 finding's verified PoC, which exercises the same `WriteParquet(nil, path, nil)` call path and confirms it produces a valid file rather than crashing.

### Lesson Learned

When hypothesizing about nil-schema behavior in parquet-go, trace the actual library code rather than assuming nil will cause a crash. The library's `NewParquetWriter` explicitly guards against nil obj (line 83), and the `Flush`/`WriteStop` path is safe when no data has been written because all loops iterate over empty collections. A confirmed finding (success/003) exercising the same nil-schema WriteParquet call already demonstrated non-crashing behavior.
