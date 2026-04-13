# H002: Transaction Parquet export crashes on non-empty archived Soroban entry lists

**Date**: 2026-04-13
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: empty results from broken control flow
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For Soroban transactions whose footprint extension contains archived entry
indexes, `export_transactions --write-parquet` should preserve those integers in
`soroban_resources_archived_entries` and continue writing the rest of the
transaction rows.

## Mechanism

`TransformTransaction()` builds `SorobanResourcesArchivedEntries` as
`[]uint32`, and `TransactionOutputParquet` keeps that field as `[]uint32` with a
repeated `INT32` tag. parquet-go accepts the schema, but when a row contains a
non-empty slice it tries to read those elements through the signed-int path and
fails with `reflect: call of reflect.Value.Int on uint32 Value`, aborting the
Parquet write on the first affected transaction.

## Trigger

1. Export any Soroban transaction whose `sorobanData.Ext.ResourceExt.ArchivedSorobanEntries`
   is non-empty.
2. Run `stellar-etl export_transactions --write-parquet ...`.
3. The Parquet writer initialization succeeds, but `writer.Write(record.ToParquet())`
   fails once it reaches that row.

## Target Code

- `internal/transform/transaction.go:171-174` — copies archived entry indexes
  into `outputSorobanArchivedEntries`
- `internal/transform/schema_parquet.go:59-66` — declares
  `SorobanResourcesArchivedEntries []uint32` as repeated `INT32`
- `internal/transform/parquet_converter.go:84-95` — forwards the `[]uint32`
  slice unchanged into the Parquet struct
- `cmd/command_utils.go:175-177` — `WriteParquet()` fatals on the first record
  write error

## Evidence

In-repo validation shows that a sample `TransactionOutputParquet` row with
non-empty `ExtraSigners` writes successfully, while the same row with only
`SorobanResourcesArchivedEntries = []uint32{1,2}` fails with
`reflect: call of reflect.Value.Int on uint32 Value`. That isolates the write
failure to the archived-entry slice rather than to the other repeated fields.

## Anti-Evidence

Transactions with an empty archived-entry list still Parquet-export correctly,
so this does not break every Soroban row. The hypothesis depends on legitimate
chain data reaching the archived-footprint path, but the source XDR field is
already wired through `TransformTransaction()`, so the trigger is real rather
than theoretical.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Fail entry 030 (condensed in summary.md) discussed the same field but a different mechanism (signed/unsigned value interpretation at high magnitudes), not the runtime reflect panic traced here.

### Trace Summary

`TransformTransaction()` populates `SorobanResourcesArchivedEntries` as `[]uint32` (transaction.go:172-173). The parquet converter passes this slice through unchanged (parquet_converter.go:94). The Parquet struct declares it as `[]uint32` with tag `type=INT32, repetitiontype=REPEATED` (schema_parquet.go:66). During marshaling, `ParquetSlice.Marshal` (parquet-go marshal.go:126-153) decomposes the slice into individual `uint32` elements. Each element reaches `InterfaceToParquetType` (types.go:223-228), where the `INT32` case tries `src.(int32)` (fails — `uint32 ≠ int32`), then calls `int32(reflect.ValueOf(src).Int())` which panics because `reflect.Value.Int()` rejects unsigned integer kinds.

### Code Paths Examined

- `internal/transform/transaction.go:141,172-173` — `outputSorobanArchivedEntries` built as `[]uint32` by casting each `ArchivedSorobanEntries` element to `uint32`
- `internal/transform/schema_parquet.go:66` — `SorobanResourcesArchivedEntries []uint32` tagged `type=INT32, repetitiontype=REPEATED`
- `internal/transform/parquet_converter.go:94` — directly assigns `to.SorobanResourcesArchivedEntries` with no type conversion
- `parquet-go/marshal/marshal.go:126-153` — `ParquetSlice.Marshal` decomposes slice into individual reflect.Value nodes
- `parquet-go/marshal/marshal.go:266-273` — leaf elements dispatched through `types.InterfaceToParquetType()`
- `parquet-go/types/types.go:223-228` — `INT32` case: `src.(int32)` assertion fails for `uint32`, falls to `reflect.ValueOf(src).Int()` which panics on unsigned kinds
- `parquet-go/marshal/marshal.go:198-208` — `recover()` catches the panic and returns it as an error
- `cmd/command_utils.go:162-180` — `WriteParquet()` calls `writer.Write()` in a loop; errors cause `cmdLogger.Fatal()`; `defer writer.WriteStop()` discards its error return

### Findings

The bug is confirmed through complete code path tracing. When a `uint32` element from the archived entries slice reaches parquet-go's `InterfaceToParquetType` with type `INT32`, `reflect.ValueOf(uint32_val).Int()` panics. The panic is recovered by `Marshal()` and returned as an error. The downstream behavior depends on buffer size:

1. **Large dataset (likely production path)**: The parquet writer's internal buffer fills during `Write()`, triggering `Flush()`. The error propagates through `flushObjs()` → `Flush()` → `Write()` → back to `WriteParquet()`, which calls `cmdLogger.Fatal("could not write record to parquet file: ", err)`. The export crashes.

2. **Small dataset**: `Write()` merely buffers the record (returns nil). The error surfaces during the deferred `WriteStop()` → `Flush(true)`, but `defer writer.WriteStop()` at command_utils.go:173 discards the error return value. The Parquet file is created but incomplete — silent data loss.

Both scenarios produce incorrect results. The `ExtraSigners []string` field works because strings go through the `BYTE_ARRAY` path, not the `INT32` path, and `[]string` uses `type=MAP, convertedtype=LIST` which takes a different marshaling route.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go` (or a new test in `cmd/`)
- **Setup**: Construct a `TransactionOutputParquet` with `SorobanResourcesArchivedEntries: []uint32{1, 2}` and all other fields at zero values. Create a local Parquet file writer and `NewParquetWriter` with the struct as schema.
- **Steps**: Call `writer.Write(parquetStruct)` then `writer.WriteStop()`. Alternatively, construct a `[]interface{}` and call `marshal.Marshal()` directly with the schema handler.
- **Assertion**: Assert that `writer.Write()` or `writer.WriteStop()` returns a non-nil error containing "reflect" or "Int on uint32". Confirm that the same struct with an empty `[]uint32{}` or `nil` slice does NOT error. The fix would be to either change the Go field to `[]int32` or add `convertedtype=UINT_32` to the parquet tag (though the latter may not resolve the reflect issue — changing the field type is the safer fix).
