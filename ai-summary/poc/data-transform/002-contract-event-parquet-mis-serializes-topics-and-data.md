# H002: Invalid Parquet schema prevents `export_contract_events --write-parquet` from creating a writer

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: complete Parquet export failure for contract events
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_contract_events --write-parquet` is invoked, the Parquet writer should successfully initialize a schema from `ContractEventOutputParquet` and produce a valid Parquet file containing contract event rows.

## Mechanism

`ContractEventOutputParquet` declares `Topics` and `TopicsDecoded` as `[]interface{}` fields with scalar `type=BYTE_ARRAY` Parquet tags but no `valuetype` metadata. When `writer.NewParquetWriter` attempts to build a schema from these tags, parquet-go's `schemahandler` tries to derive a Parquet type from an empty string (the missing `valuetype`), which fails with `"type : not a valid Type string"`. The writer cannot be created, so no contract event data is ever written to Parquet.

The failing path is: `cmd/export_contract_events.go:63-65` → `cmd/command_utils.go:169-171` (calls `WriteParquet`) → `writer.NewParquetWriter(parquetFile, new(transform.ContractEventOutputParquet), 1)` → schema creation fails.

## Trigger

Run `export_contract_events --write-parquet` on any ledger range. The command fails at writer initialization before processing any events.

## Target Code

- `internal/transform/schema_parquet.go:393-394` — `Topics` and `TopicsDecoded` are `[]interface{}` with scalar `type=BYTE_ARRAY` tags but no `valuetype`, which parquet-go rejects during schema creation.
- `cmd/export_contract_events.go:63-65` — Calls `WriteParquet` with `new(transform.ContractEventOutputParquet)`.
- `cmd/command_utils.go:169-171` — `WriteParquet` passes the schema object to `writer.NewParquetWriter`.

## Evidence

- All other `*OutputParquet` structs with scalar fields use types that parquet-go can directly map (e.g., `string`, `int64`, `bool`). `ContractEventOutputParquet` is the only struct that uses `[]interface{}` fields, and its tags do not include the `valuetype` metadata required by parquet-go for slice fields.
- A control schema (`TtlOutputParquet`) succeeds at writer creation, confirming the failure is specific to the broken tags on `ContractEventOutputParquet`.

## Anti-Evidence

None — the failure is deterministic and affects every invocation of `export_contract_events --write-parquet`.

---

## Review

**Verdict**: VIABLE
**Severity**: High → revised to Medium per final review
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete data path from `serializeScVal()`/`serializeScValArray()` through `ContractEventOutput` population in `parseDiagnosticEvent()`, through `ToParquet()` in `parquet_converter.go`, and into the `xitongsys/parquet-go` v1.6.2 schema creation path. Before any rows are ever written, the writer fails at schema creation because the `[]interface{}` fields (`Topics`, `TopicsDecoded`) have scalar `type=BYTE_ARRAY` tags without `valuetype` metadata, which parquet-go rejects.

### Code Paths Examined

- `internal/transform/schema_parquet.go:393-394` — `Topics` and `TopicsDecoded` are `[]interface{}` with scalar `type=BYTE_ARRAY` tags but no `valuetype`.
- `cmd/export_contract_events.go:63-65` — Calls `WriteParquet` with `new(transform.ContractEventOutputParquet)`.
- `cmd/command_utils.go:169-171` — `WriteParquet` passes the schema object to `writer.NewParquetWriter`.
- `github.com/xitongsys/parquet-go@v1.6.2/schema/schemahandler.go:297-335` — Schema creation fails when deriving type from empty valuetype string.
- `internal/transform/parquet_converter.go:425-442` — `ToParquet()` copies fields verbatim (never reached due to writer init failure).

### Findings

The originally hypothesized payload-stringification corruption (where `json.RawMessage` would be rendered as `"<json.RawMessage Value>"`) is unreachable because the writer fails at schema creation first. The actual bug is simpler: `ContractEventOutputParquet` has invalid Parquet struct tags for its slice fields. This makes `export_contract_events --write-parquet` completely non-functional.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go`
- **Setup**: Create a temp Parquet file and call `writer.NewParquetWriter` with `new(ContractEventOutputParquet)`.
- **Assertion**: Writer creation fails. A control schema (`TtlOutputParquet`) succeeds.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventParquetSchemaCreationFails"
**Test Language**: Go

### Demonstration

The test calls `writer.NewParquetWriter(pf, new(ContractEventOutputParquet), 1)` — the same call made by `cmd.WriteParquet` when `--write-parquet` is used — and confirms it fails with `"failed to create schema from tag map: type : not a valid Type string"`. This proves that `export_contract_events --write-parquet` cannot even create a Parquet writer, so no contract event data can be exported to Parquet format at all. A control schema (`TtlOutputParquet`) succeeds, confirming the failure is specific to the broken `[]interface{}` tags on `ContractEventOutputParquet`.

### Test Body

```go
package transform

import (
	"os"
	"strings"
	"testing"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

// TestContractEventParquetSchemaCreationFails demonstrates that
// ContractEventOutputParquet has invalid Parquet tags on its Topics
// and TopicsDecoded []interface{} fields. The parquet-go writer cannot
// create a schema from these tags, so export_contract_events --write-parquet
// fails at writer initialization and never produces any output.
func TestContractEventParquetSchemaCreationFails(t *testing.T) {
	tmpFile, err := os.CreateTemp("", "poc-contract-event-*.parquet")
	if err != nil {
		t.Fatalf("failed to create temp file: %v", err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("failed to create local file writer: %v", err)
	}
	defer pf.Close()

	// This is the exact call made by cmd.WriteParquet for contract events.
	// It fails because Topics/TopicsDecoded are []interface{} with scalar
	// type=BYTE_ARRAY tags but no valuetype — parquet-go rejects the schema.
	pw, err := writer.NewParquetWriter(pf, new(ContractEventOutputParquet), 1)
	if pw != nil {
		pw.WriteStop()
	}

	if err == nil {
		t.Fatal("expected NewParquetWriter to fail for ContractEventOutputParquet, but it succeeded")
	}

	t.Logf("NewParquetWriter error: %v", err)

	if !strings.Contains(err.Error(), "not a valid Type") {
		t.Logf("Warning: error message does not contain 'not a valid Type': %s", err.Error())
	}

	// Control: verify a known-good Parquet schema succeeds, confirming
	// the failure is specific to ContractEventOutputParquet's broken tags.
	pf2, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatalf("failed to create second local file writer: %v", err)
	}
	defer pf2.Close()

	pw2, err := writer.NewParquetWriter(pf2, new(TtlOutputParquet), 1)
	if err != nil {
		t.Fatalf("TtlOutputParquet schema creation should succeed but got: %v", err)
	}
	pw2.WriteStop()

	t.Log("CONFIRMED: ContractEventOutputParquet schema creation fails; TtlOutputParquet succeeds.")
	t.Log("export_contract_events --write-parquet cannot create a Parquet writer at all.")
}
```

### Test Output

```
=== RUN   TestContractEventParquetSchemaCreationFails
    data_integrity_poc_test.go:44: NewParquetWriter error: failed to create schema from tag map: type : not a valid Type string
    data_integrity_poc_test.go:64: CONFIRMED: ContractEventOutputParquet schema creation fails; TtlOutputParquet succeeds.
    data_integrity_poc_test.go:65: export_contract_events --write-parquet cannot create a Parquet writer at all.
--- PASS: TestContractEventParquetSchemaCreationFails (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.827s
```
