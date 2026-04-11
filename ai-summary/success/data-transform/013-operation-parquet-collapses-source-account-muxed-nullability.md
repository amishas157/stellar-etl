# 013: Operation Parquet rewrites absent `source_account_muxed` to empty strings

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` only sets `source_account_muxed` when the effective source account is actually muxed. The JSON/export struct preserves that absence with `omitempty`, but the Parquet path forces the field into a required `string` and writes `""` for ordinary `G...` accounts.

This is a real structural corruption bug in normal export flow. `export_operations --write-parquet` collects `OperationOutput` rows and writes `OperationOutputParquet` via `ToParquet()`, so downstream Parquet consumers lose null semantics and cannot distinguish "no muxed source account" from a concrete empty-string value.

## Root Cause

`TransformOperation()` builds a `null.String` for muxed source metadata and only populates it for `KEY_TYPE_MUXED_ED25519`. It then discards that nullability by assigning `.String` into `OperationOutput.SourceAccountMuxed`, and `OperationOutputParquet` narrows the field again to a required `string`, so `ToParquet()` serializes non-muxed rows as `source_account_muxed=""`.

## Reproduction

Run `export_operations --write-parquet` on any operation whose effective source account is a regular ed25519 `G...` account. The JSON representation omits `source_account_muxed`; after Parquet conversion, the same row contains `source_account_muxed=""`.

## Affected Code

- `internal/transform/operation.go:TransformOperation:30-100` — populates muxed source metadata only for muxed accounts, then stores `.String` in `OperationOutput`.
- `internal/transform/operation.go:getOperationSourceAccount:287-294` — resolves the effective source account from the operation or transaction envelope.
- `internal/transform/schema.go:OperationOutput:137-149` — JSON schema models `source_account_muxed` as omittable.
- `internal/transform/schema_parquet.go:OperationOutputParquet:117-129` — Parquet schema narrows `source_account_muxed` to a required `string`.
- `internal/transform/parquet_converter.go:OperationOutput.ToParquet:147-160` — converter copies the empty string directly into the Parquet row.
- `cmd/export_operations.go:25-66` — production export path appends `OperationOutput` rows and writes them as `OperationOutputParquet`.
- `cmd/command_utils.go:162-176` — Parquet writer emits each row via `record.ToParquet()`.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestOperationParquetFlattensSourceAccountMuxedToEmptyString`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestOperationParquetFlattensSourceAccountMuxedToEmptyString -v`
  4. Observe the JSON layer omit `source_account_muxed` while the Parquet row contains `""`

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"
)

func TestOperationParquetFlattensSourceAccountMuxedToEmptyString(t *testing.T) {
	operationOutput, err := TransformOperation(
		genericBumpOperation,
		0,
		genericLedgerTransaction,
		1,
		makeLedgerCloseMeta(),
		"",
	)
	if err != nil {
		t.Fatalf("TransformOperation failed: %v", err)
	}

	if operationOutput.SourceAccountMuxed != "" {
		t.Fatalf("expected empty muxed source account for non-muxed input, got %q", operationOutput.SourceAccountMuxed)
	}

	jsonBytes, err := json.Marshal(operationOutput)
	if err != nil {
		t.Fatalf("marshal operation output: %v", err)
	}

	var jsonMap map[string]interface{}
	if err := json.Unmarshal(jsonBytes, &jsonMap); err != nil {
		t.Fatalf("unmarshal operation output: %v", err)
	}

	if _, ok := jsonMap["source_account_muxed"]; ok {
		t.Fatalf("expected JSON to omit source_account_muxed for non-muxed source account, got %v", jsonMap["source_account_muxed"])
	}

	parquetOutput := operationOutput.ToParquet().(OperationOutputParquet)
	if parquetOutput.SourceAccountMuxed != "" {
		t.Fatalf("expected parquet source_account_muxed to be serialized as empty string, got %q", parquetOutput.SourceAccountMuxed)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Non-muxed operations should preserve the absence of muxed metadata in Parquet so `source_account_muxed` remains null or otherwise distinguishable from a concrete value.
- **Actual**: The Parquet export fabricates `source_account_muxed=""` for ordinary non-muxed operations.

## Adversarial Review

1. Exercises claimed bug: YES — the validated PoC uses the package's real non-muxed operation fixture, runs `TransformOperation()`, marshals the real output to JSON, then runs the production `ToParquet()` converter.
2. Realistic preconditions: YES — ordinary `G...` source accounts are the default Stellar case, and the built-in fixture uses a standard ed25519 muxed-account arm with no muxed ID.
3. Bug vs by-design: BUG — the ETL already chose absent semantics for this field in the JSON/export layer via conditional population plus `omitempty`; only the Parquet layer discards that information.
4. Final severity: High — this is silent structural corruption of non-financial operation metadata that breaks null-sensitive attribution and analytics.
5. In scope: YES — it is a concrete transform/export path that produces plausible but wrong Parquet data during normal operation.
6. Test correctness: CORRECT — the test proves the field is absent in the real JSON representation before conversion and present as a fabricated empty string only after Parquet conversion.
7. Alternative explanations: NONE — while `""` is not a valid muxed address, it is still non-null data and changes downstream query semantics (`IS NULL`, null counts, joins, and filters).
8. Novelty: NOVEL

## Suggested Fix

Make `OperationOutput.SourceAccountMuxed` and `OperationOutputParquet.SourceAccountMuxed` nullable, such as `null.String` / `*string` with an optional Parquet tag, and emit `nil` when the effective source account is not muxed. Audit sibling muxed-account fields across the Parquet schemas for the same nullable-to-required-string flattening pattern.
