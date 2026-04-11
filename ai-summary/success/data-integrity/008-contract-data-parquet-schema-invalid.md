# 008: Contract data Parquet schema is invalid

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes --export-contract-data --write-parquet` cannot initialize its Parquet writer. The production `ContractDataOutputParquet` schema declares four dynamic payload fields as `interface{}` with `type=MAP`, and parquet-go rejects that schema before any rows are written.

This is a real export failure, not just a bad PoC. The exact writer-construction path used by `cmd.WriteParquet()` fails with `failed to create schema from tag map: type MAP: not a valid Type string`, so contract-data Parquet output is never produced for any non-nonce contract-data row.

## Root Cause

`TransformContractData()` fills `Key` and `Val` with base64-encoded XDR strings and `KeyDecoded` and `ValDecoded` with decoded JSON payloads from `serializeScVal()`. But `ContractDataOutputParquet` models all four fields as `interface{}` columns tagged as Parquet `MAP`s. When `writer.NewParquetWriter(...)` derives the schema from that struct, parquet-go inspects the Go field kind, sees `interface{}` instead of a concrete map/list layout, and aborts schema creation with `type MAP: not a valid Type string`.

The failure is therefore rooted in the production schema definition itself, not in any particular contract-data row value.

## Reproduction

Run a normal ledger-entry-changes export with both `--export-contract-data` and `--write-parquet`. Once the command accumulates at least one exportable contract-data row, `cmd/export_ledger_entry_changes.go` selects `new(transform.ContractDataOutputParquet)` and passes it to `WriteParquet(...)`. Writer initialization fails immediately, and the command fatals before the first Parquet row is written.

## Affected Code

- `internal/transform/schema_parquet.go:260-281` — contract-data Parquet schema declares `key`, `key_decoded`, `val`, and `val_decoded` as `interface{}` fields tagged as `MAP`
- `internal/transform/parquet_converter.go:292-314` — converter copies the dynamic payloads straight into those invalid Parquet fields
- `internal/transform/contract_data.go:121-126` — contract-data transformation populates the payloads through `serializeScVal()`
- `internal/transform/contract_events.go:96-119` — `serializeScVal()` returns base64 strings plus decoded JSON, not a concrete Parquet map layout
- `cmd/export_ledger_entry_changes.go:344-347` — contract-data export selects `ContractDataOutputParquet`
- `cmd/export_ledger_entry_changes.go:370-372` — the export path calls `WriteParquet(...)` for that schema
- `cmd/command_utils.go:162-171` — `WriteParquet()` fatals when parquet-go rejects schema creation

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractDataParquetSchemaInvalid`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Run: `go test ./internal/transform/... -run TestContractDataParquetSchemaInvalid -v`
  4. Observe: Parquet writer creation fails with `type MAP: not a valid Type string` instead of writing any contract-data rows.

### Test Body

```go
func TestContractDataParquetSchemaInvalid(t *testing.T) {
	parquetPath := filepath.Join(t.TempDir(), "contract-data.parquet")

	parquetFile, err := local.NewLocalFileWriter(parquetPath)
	if err != nil {
		t.Fatalf("failed to create parquet file: %v", err)
	}
	defer parquetFile.Close()

	_, err = writer.NewParquetWriter(parquetFile, new(ContractDataOutputParquet), 1)
	if err == nil {
		t.Fatal("expected contract data parquet writer creation to fail, but it succeeded")
	}

	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("expected schema error containing %q, got: %v", "not a valid Type string", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_ledger_entry_changes --export-contract-data --write-parquet` should construct a valid Parquet writer and emit contract-data rows that preserve the same payloads already exported in JSON.
- **Actual**: Parquet writer initialization fails immediately with `failed to create schema from tag map: type MAP: not a valid Type string`, so no contract-data Parquet rows are written.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `writer.NewParquetWriter(...)`, the same writer-construction path used by `cmd.WriteParquet()`.
2. Realistic preconditions: YES — the only preconditions are enabling the documented contract-data export and Parquet output path on a batch that contains an exportable contract-data row.
3. Bug vs by-design: BUG — the exporter is clearly intended to support contract-data Parquet output, and this schema prevents any output from being produced.
4. Final severity: Medium — this is a concrete export failure and data-loss condition for one output mode, not silent corruption of an already-written financial field.
5. In scope: YES — it is a production code path that aborts normal export behavior.
6. Test correctness: CORRECT — the test checks the real writer initialization failure rather than asserting on mocks or on values it set up itself.
7. Alternative explanations: NONE — the failure happens before any row write and is caused by the production schema declaration alone.
8. Novelty: NOVEL

## Suggested Fix

Replace the four dynamic contract-data Parquet fields with concrete Parquet-friendly types. The simplest safe fix is to serialize `key`, `key_decoded`, `val`, and `val_decoded` to explicit strings or byte slices before they reach the Parquet schema, then update `ContractDataOutputParquet` and `ContractDataOutput.ToParquet()` to match those concrete types.
