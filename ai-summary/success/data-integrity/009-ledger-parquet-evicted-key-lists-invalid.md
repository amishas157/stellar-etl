# 009: Ledger Parquet evicted-key string lists use an invalid schema

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`export_ledgers --write-parquet` cannot initialize its Parquet writer. The production `LedgerOutputParquet` schema declares `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` as `[]string`, but the Parquet tags omit the list element type that parquet-go requires for slice fields.

This is a real export failure, not a PoC artifact. The exact writer-construction path used by `cmd.WriteParquet()` fails with `failed to create schema from tag map: type : not a valid Type string`, so ledger Parquet output is never produced once the Parquet path is enabled.

## Root Cause

`LedgerOutputParquet` models the two evicted-key columns as `[]string` and tags them like scalar UTF-8 values:

- `type=BYTE_ARRAY, convertedtype=UTF8`

For slice fields, parquet-go does not infer the child element type from Go's `[]string` alone in the LIST branch it uses here. Instead, it calls `GetValueTagMap(item.Info)`, which copies only `ValueType`/`ValueConvertedType` into the child element schema. Because these tags never set `valuetype=BYTE_ARRAY`, the derived child type is empty and schema creation aborts with `not a valid Type string`.

The ledger export path always passes `new(transform.LedgerOutputParquet)` into `WriteParquet(...)`, so this bad schema prevents writer initialization before any row is written, even when the evicted-key arrays would be empty for the current batch.

## Reproduction

Run ledger export with `--write-parquet`. After JSON rows are emitted, `cmd/export_ledgers.go` calls `WriteParquet(..., new(transform.LedgerOutputParquet))`. During writer construction, parquet-go reflects on `LedgerOutputParquet`, reaches the two `[]string` fields, derives an empty list element type, and fails before the first Parquet row is written.

## Affected Code

- `internal/transform/schema_parquet.go:27-28` — ledger Parquet schema declares `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` as `[]string` without `valuetype`
- `internal/transform/schema_parquet.go:59` — `ExtraSigners` shows the working repeated-string tag pattern used elsewhere
- `cmd/export_ledgers.go:70-72` — ledger export selects `LedgerOutputParquet` and calls `WriteParquet(...)`
- `cmd/command_utils.go:162-171` — `WriteParquet()` constructs the parquet-go writer and fatals on schema creation errors
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/common/common.go:553-567` — `GetValueTagMap()` copies only `ValueType`/`ValueConvertedType` into the list element tag
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/schema/schemahandler.go:323-334` — slice fields use `GetValueTagMap(item.Info)` when building the LIST child element
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/schema/schemahandler.go:388-391` — schema creation fails when the derived child type is empty

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestLedgerParquetEvictedKeyListsInvalid`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `internal/transform/data_integrity_poc_test.go`
  3. Paste the code below into that file
  4. Run: `go test ./internal/transform/... -run TestLedgerParquetEvictedKeyListsInvalid -v`
  5. Observe: schema creation and Parquet writer initialization both fail with `not a valid Type string` instead of producing a ledger Parquet writer.

### Test Body

```go
package transform

import (
	"path/filepath"
	"strings"
	"testing"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/schema"
	"github.com/xitongsys/parquet-go/writer"
)

func TestLedgerParquetEvictedKeyListsInvalid(t *testing.T) {
	_, err := schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))
	if err == nil {
		t.Fatal("expected schema creation to fail for LedgerOutputParquet")
	}
	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("expected invalid type error, got: %v", err)
	}

	outputPath := filepath.Join(t.TempDir(), "ledgers.parquet")
	fileWriter, err := local.NewLocalFileWriter(outputPath)
	if err != nil {
		t.Fatalf("creating parquet file: %v", err)
	}
	defer fileWriter.Close()

	_, err = writer.NewParquetWriter(fileWriter, new(LedgerOutputParquet), 1)
	if err == nil {
		t.Fatal("expected parquet writer creation to fail for LedgerOutputParquet")
	}
	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("expected invalid type error from parquet writer, got: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `export_ledgers --write-parquet` should construct a valid Parquet writer and emit ledger rows that preserve `evicted_ledger_keys_type` and `evicted_ledger_keys_hash`.
- **Actual**: Parquet writer initialization fails immediately with `failed to create schema from tag map: type : not a valid Type string`, so no ledger Parquet rows are written.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls both `schema.NewSchemaHandlerFromStruct(...)` and `writer.NewParquetWriter(...)`, and the latter is the same writer-construction path used by `cmd.WriteParquet()`.
2. Realistic preconditions: YES — the only precondition is enabling the documented `--write-parquet` path on `export_ledgers`.
3. Bug vs by-design: BUG — ledger Parquet export is clearly intended to work, and `TransactionOutputParquet.ExtraSigners` shows the codebase already knows the correct repeated-string tag pattern.
4. Final severity: Medium — this is a concrete export failure and data-loss condition for one output mode, not silent corruption of an already-written financial field.
5. In scope: YES — it is a production export path that fails under normal operation.
6. Test correctness: CORRECT — the test checks real schema and writer initialization, with no mocks and no tautological assertions.
7. Alternative explanations: NONE — the failure occurs before any rows are written and is caused by the production Parquet schema declaration itself.
8. Novelty: NOVEL

## Suggested Fix

Change both ledger evicted-key fields to use the same repeated-string tag pattern already used by `TransactionOutputParquet.ExtraSigners`, e.g.:

```go
EvictedLedgerKeysType []string `parquet:"name=evicted_ledger_keys_type, type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8"`
EvictedLedgerKeysHash []string `parquet:"name=evicted_ledger_keys_hash, type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8"`
```
