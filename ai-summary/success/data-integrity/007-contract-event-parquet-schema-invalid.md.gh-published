# 007: Contract event Parquet schema is invalid

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_contract_events --write-parquet` cannot initialize its Parquet writer. The production `ContractEventOutputParquet` schema contains repeated fields tagged as scalar `BYTE_ARRAY` columns, and parquet-go rejects that schema before any rows are written.

This is a real export failure, not just a bad PoC. The exact writer construction path used by `cmd.WriteParquet()` fails with `failed to create schema from tag map: type : not a valid Type string`, so contract-event Parquet output is never produced.

## Root Cause

`ContractEventOutputParquet` declares `Topics` and `TopicsDecoded` as repeated values (`[]interface{}`) but tags them like scalar UTF-8 columns (`type=BYTE_ARRAY, convertedtype=UTF8`). When `writer.NewParquetWriter(...)` derives the schema, parquet-go treats those fields as repeated/list values and expects an element type annotation; instead it receives an empty element type and aborts schema creation with `type : not a valid Type string`.

This bad tag pattern is broader than contract events alone, but contract-event export independently hits it through its own production schema and fails every time the Parquet path is enabled.

## Reproduction

Run any contract-event export with `--write-parquet`. After JSON rows are emitted, the command reaches `WriteParquet(..., new(transform.ContractEventOutputParquet))`, and writer initialization fails before the first Parquet row is written.

## Affected Code

- `cmd/export_contract_events.go:63-64` — contract-event export selects `ContractEventOutputParquet` for the Parquet path
- `cmd/command_utils.go:162-171` — `WriteParquet()` constructs the parquet-go writer and fatals on schema creation errors
- `internal/transform/schema_parquet.go:382-399` — contract-event Parquet schema declares repeated dynamic fields with incompatible parquet tags

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventParquetSchemaInvalid`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, add the needed parquet imports if absent, then run `go test ./internal/transform/... -run TestContractEventParquetSchemaInvalid -v`.

### Test Body

```go
func TestContractEventParquetSchemaInvalid(t *testing.T) {
	parquetPath := filepath.Join(t.TempDir(), "contract-events.parquet")

	parquetFile, err := local.NewLocalFileWriter(parquetPath)
	if err != nil {
		t.Fatalf("failed to create parquet file: %v", err)
	}
	defer parquetFile.Close()

	_, err = writer.NewParquetWriter(parquetFile, new(ContractEventOutputParquet), 1)
	if err == nil {
		t.Fatal("expected contract event parquet writer creation to fail, but it succeeded")
	}

	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("expected schema error containing %q, got: %v", "not a valid Type string", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_contract_events --write-parquet` should create a valid Parquet writer and emit contract-event rows.
- **Actual**: Parquet writer initialization fails immediately with `failed to create schema from tag map: type : not a valid Type string`, so no contract-event Parquet rows are written.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls `writer.NewParquetWriter(...)`, the same writer-construction path used by `cmd.WriteParquet()`.
2. Realistic preconditions: YES — the only precondition is enabling the documented `--write-parquet` path on `export_contract_events`.
3. Bug vs by-design: BUG — Parquet export is intended to work, and this schema prevents the exporter from producing any Parquet output.
4. Final severity: Medium — this is an export failure/data-loss condition for one output mode, not silent numeric corruption of an already-written field.
5. In scope: YES — it is a concrete production code path that produces missing output under normal operation.
6. Test correctness: CORRECT — the assertion checks the exact writer initialization error instead of asserting on mocked behavior.
7. Alternative explanations: NONE — the failure occurs before any row data is written and is caused by the production schema definition itself; finding the same tag-pattern bug in another schema does not negate the contract-event failure.
8. Novelty: NOVEL

## Suggested Fix

Replace the repeated contract-event fields with concrete Parquet-friendly types and use valid repeated/list tags that specify the element type. If the payloads must remain dynamic at the JSON layer, serialize them to explicit strings or byte slices before they reach the Parquet schema.
