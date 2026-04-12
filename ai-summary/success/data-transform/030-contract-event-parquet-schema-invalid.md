# 030: Contract-event Parquet writer initialization fails on invalid repeated-field tags

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`export_contract_events --write-parquet` cannot initialize its Parquet writer, so the command never produces a contract-event Parquet artifact. The failure is deterministic: `ContractEventOutputParquet` models `topics` and `topics_decoded` as repeated fields but omits the element `valuetype` metadata that parquet-go requires to derive the schema.

## Root Cause

`ContractEventOutputParquet` declares `Topics` and `TopicsDecoded` as `[]interface{}` while tagging them like scalar UTF-8 columns. In parquet-go's slice handling path, repeated/list fields derive their element schema from `valuetype`; because these tags omit it, schema creation reaches `TypeFromString("")` and aborts before any row-writing logic runs.

## Reproduction

1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
3. Run `go test ./internal/transform/... -run TestContractEventParquetSchemaCreationFails -v`.
4. Observe that `writer.NewParquetWriter(...)` fails with `failed to create schema from tag map: type : not a valid Type string` instead of producing a contract-event Parquet writer.

## Affected Code

- `internal/transform/schema_parquet.go:382-399` — `ContractEventOutputParquet` tags `Topics` and `TopicsDecoded` as repeated data without a Parquet element `valuetype`.
- `cmd/export_contract_events.go:63-65` — the contract-event export command always passes `new(transform.ContractEventOutputParquet)` into the shared Parquet writer path when `--write-parquet` is enabled.
- `cmd/command_utils.go:162-173` — `WriteParquet()` calls `writer.NewParquetWriter(...)`, so schema construction fails before any rows are written.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestContractEventParquetSchemaCreationFails`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"path/filepath"
	"strings"
	"testing"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

func TestContractEventParquetSchemaCreationFails(t *testing.T) {
	dir := t.TempDir()

	badWriter, err := local.NewLocalFileWriter(filepath.Join(dir, "contract-events.parquet"))
	if err != nil {
		t.Fatalf("creating local parquet writer: %v", err)
	}
	defer badWriter.Close()

	_, err = writer.NewParquetWriter(badWriter, new(ContractEventOutputParquet), 1)
	if err == nil {
		t.Fatal("expected ContractEventOutputParquet schema creation to fail")
	}
	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("expected schema error containing %q, got %v", "not a valid Type string", err)
	}

	controlFile, err := local.NewLocalFileWriter(filepath.Join(dir, "ttl.parquet"))
	if err != nil {
		t.Fatalf("creating control parquet writer: %v", err)
	}
	defer controlFile.Close()

	controlWriter, err := writer.NewParquetWriter(controlFile, new(TtlOutputParquet), 1)
	if err != nil {
		t.Fatalf("expected control schema to initialize, got %v", err)
	}
	if err := controlWriter.WriteStop(); err != nil {
		t.Fatalf("stopping control parquet writer: %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Enabling `--write-parquet` should initialize a Parquet writer and emit contract-event rows.
- **Actual**: Parquet writer initialization fails immediately with `failed to create schema from tag map: type : not a valid Type string`, so no contract-event Parquet rows are written.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `writer.NewParquetWriter(...)`, the same writer-construction path reached by `cmd.WriteParquet()` from `export_contract_events`.
2. Realistic preconditions: YES — the only prerequisite is enabling the documented `--write-parquet` mode on a normal contract-event export.
3. Bug vs by-design: BUG — the command advertises Parquet export support, and the schema comment says this struct is the Parquet representation of contract events.
4. Final severity: Medium — this is a deterministic export failure for one output mode, not silent monetary corruption.
5. In scope: YES — it is a concrete production code path that prevents requested output from being generated.
6. Test correctness: CORRECT — the test uses production types, no mocks, and a control schema to show parquet-go itself works when the schema tags are valid.
7. Alternative explanations: NONE — parquet-go's slice schema path reads `valuetype` for list elements, and these tags leave it empty.
8. Novelty: NOT ASSESSED HERE

## Suggested Fix

Model the repeated contract-event topic fields with valid Parquet list metadata, for example by using a concrete element type and `valuetype=BYTE_ARRAY, valueconvertedtype=UTF8`, or by serializing the dynamic arrays into a supported concrete representation before Parquet export.
