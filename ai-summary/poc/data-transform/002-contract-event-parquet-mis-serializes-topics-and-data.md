# H002: contract-event Parquet conversion pushes arrays and JSON blobs into scalar UTF-8 columns

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_contract_events --write-parquet` serializes `topics`, `topics_decoded`, `data`, and `data_decoded`, the Parquet row should preserve the same logical values as the JSON export. For array/object-shaped values, that means either using a repeated/nested Parquet schema or first converting them to canonical JSON strings before storing them in scalar UTF-8 columns.

## Mechanism

`serializeScValArray()` builds `Topics` as a Go `[]interface{}` and `serializeScVal()` produces decoded payloads as `json.RawMessage`, but `ContractEventOutputParquet` declares those columns as scalar `BYTE_ARRAY UTF8` fields. `ContractEventOutput.ToParquet()` then copies the raw slice/interface values directly instead of JSON-encoding them like the operation/effect/config-setting converters do, so multi-topic rows and decoded payloads are handed to the Parquet writer with a type/shape mismatch and can be stringified incorrectly or written as malformed scalar text.

## Trigger

Run `export_contract_events --write-parquet` on any ledger that contains a contract event with one or more topics and decoded SCVal payloads. Compare the JSON row's array/object values with the Parquet row produced from the same `ContractEventOutput`; the Parquet path will not preserve those fields as canonical JSON arrays/objects because it sends Go slices/interfaces into scalar UTF-8 columns.

## Target Code

- `internal/transform/contract_events.go:96-137` — `serializeScVal()` and `serializeScValArray()` produce strings, `json.RawMessage`, and `[]interface{}` values for event payloads.
- `internal/transform/schema_parquet.go:383-399` — `ContractEventOutputParquet` defines `topics`, `topics_decoded`, `data`, and `data_decoded` as scalar UTF-8 columns.
- `internal/transform/parquet_converter.go:425-441` — `ContractEventOutput.ToParquet()` copies those dynamic values directly instead of normalizing them with `toJSONString()`.

## Evidence

Sibling converters with dynamic payloads (`OperationOutput`, `EffectOutput`, `ConfigSettingOutput`) explicitly call `toJSONString()` before assigning into scalar UTF-8 Parquet columns. The contract-event converter is the outlier: it passes `[]interface{}` and interface-typed JSON blobs straight through even though the Parquet schema is not declared as repeated/nested JSON.

## Anti-Evidence

If the underlying Parquet library happens to stringify these interface values in a stable way, the export may still complete. But even then, the output would still depend on Go's runtime formatting of slices/interfaces rather than the canonical JSON representation already produced at the transform layer.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete data path from `serializeScVal()`/`serializeScValArray()` through `ContractEventOutput` population in `parseDiagnosticEvent()`, through `ToParquet()` in `parquet_converter.go`, and into the `xitongsys/parquet-go` v1.6.2 marshal path. The `TopicsDecoded` and `DataDecoded` fields contain `json.RawMessage` (which is `[]byte`) values wrapped in `interface{}`. When the parquet-go marshaler reaches these leaf values, it calls `InterfaceToParquetType()` which tries `src.(string)` — this fails for `json.RawMessage` — then falls back to `reflect.ValueOf(src).String()`. Per Go's `reflect.Value.String()` contract, this returns the literal string `"<json.RawMessage Value>"` for non-string types, completely destroying the decoded payload data.

### Code Paths Examined

- `internal/transform/contract_events.go:96-119` — `serializeScVal()` returns `(string, json.RawMessage)` for data/dataDecoded respectively. The base64-encoded `serializedData` is always a `string`, but `serializedDataDecoded` is a `json.RawMessage` (`[]byte`).
- `internal/transform/contract_events.go:122-137` — `serializeScValArray()` aggregates into `[]interface{}` slices. `data` elements are strings (base64), `dataDecoded` elements are `json.RawMessage`.
- `internal/transform/contract_events.go:189-241` — `parseDiagnosticEvent()` assigns Topics/TopicsDecoded/Data/DataDecoded directly to the output struct.
- `internal/transform/parquet_converter.go:425-442` — `ToParquet()` copies fields verbatim: `Topics: ceo.Topics`, `TopicsDecoded: ceo.TopicsDecoded`, `Data: ceo.Data`, `DataDecoded: ceo.DataDecoded` — no `toJSONString()` normalization.
- `internal/transform/parquet_converter.go:147-161` — Contrast: `OperationOutput.ToParquet()` calls `toJSONString(oo.OperationDetails)` for its dynamic field.
- `internal/transform/parquet_converter.go:277-290` — Contrast: `EffectOutput.ToParquet()` calls `toJSONString(eo.Details)`.
- `internal/transform/parquet_converter.go:339-411` — Contrast: `ConfigSettingOutput.ToParquet()` calls `toJSONString(cso.ContractCostParamsCpuInsns)`.
- `github.com/xitongsys/parquet-go@v1.6.2/types/types.go:206-265` — `InterfaceToParquetType()`: for BYTE_ARRAY type, tries `src.(string)`, on failure calls `reflect.ValueOf(src).String()` which returns `"<json.RawMessage Value>"` for non-string kinds.
- `github.com/xitongsys/parquet-go@v1.6.2/marshal/marshal.go:244-274` — Main marshal loop: `interface{}` kind fields fall to the else-branch (not Ptr/Struct/Slice/Map), calling `InterfaceToParquetType` directly on the unwrapped interface value.

### Findings

**Scope refinement**: The hypothesis correctly identifies the missing `toJSONString()` normalization, but the actual corruption scope is narrower than the four fields claimed:

| Field | Runtime Type | Parquet Result | Status |
|-------|-------------|----------------|--------|
| `Topics` | `[]interface{}` of `string` (base64) | Each string written correctly as repeated BYTE_ARRAY via `ParquetSlice` → `InterfaceToParquetType` string path | ✅ Correct |
| `TopicsDecoded` | `[]interface{}` of `json.RawMessage` | Each element → `InterfaceToParquetType` → `reflect.ValueOf(rawMsg).String()` → `"<json.RawMessage Value>"` | ❌ **CORRUPTED** |
| `Data` | `interface{}` holding `string` (base64) | `InterfaceToParquetType` string path works | ✅ Correct |
| `DataDecoded` | `interface{}` holding `json.RawMessage` | `InterfaceToParquetType` → `reflect.ValueOf(rawMsg).String()` → `"<json.RawMessage Value>"` | ❌ **CORRUPTED** |

The corruption mechanism is `reflect.Value.String()` on a non-string Kind returning `"<T Value>"` per Go specification. Every contract event with a valid decoded payload produces the literal string `"<json.RawMessage Value>"` instead of the actual decoded JSON content, losing all semantic information.

**Impact**: Every Parquet-exported contract event row has its `topics_decoded` and `data_decoded` columns filled with the useless sentinel `"<json.RawMessage Value>"`. Downstream consumers querying decoded event payloads from Parquet get zero useful data. The JSON export path is unaffected (json.Marshal handles json.RawMessage correctly).

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (create if not present, or append to existing contract_events_test.go)
- **Setup**: Construct a `ContractEventOutput` with Topics containing base64 strings, TopicsDecoded containing `json.RawMessage` values (e.g., `json.RawMessage('{"type":"Address","value":"GABC..."}')`), Data as a base64 string, and DataDecoded as a `json.RawMessage`.
- **Steps**: Call `contractEventOutput.ToParquet()` to get the `ContractEventOutputParquet` struct. Inspect the `TopicsDecoded` and `DataDecoded` fields.
- **Assertion**: Assert that `TopicsDecoded` elements are valid JSON strings (not `"<json.RawMessage Value>"`). Assert `DataDecoded` contains the original JSON content. Compare against `toJSONString()` output to show the expected normalization. The test should fail on current code and pass after adding `toJSONString()` calls in `ToParquet()`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractEventParquetWriterFailsAtSchemaCreation"
**Test Language**: Go

### Demonstration

The test exercises the actual Parquet writer creation path — the same call made by `cmd.WriteParquet` when `--write-parquet` is used. It calls `writer.NewParquetWriter(parquetFile, new(ContractEventOutputParquet), 1)` and demonstrates that schema creation fails with error `"failed to create schema from tag map: type : not a valid Type string"`. This means `export_contract_events --write-parquet` is completely broken: it cannot even create a Parquet writer, so no contract event data can be exported to Parquet format at all. The root cause is that the `Topics` and `TopicsDecoded` fields are declared as `[]interface{}` with scalar `type=BYTE_ARRAY` tags but no `valuetype` — parquet-go requires `valuetype` for slice fields that aren't marked `repetitiontype=REPEATED`. A control schema (`TtlOutputParquet`, which has no slice fields) succeeds, confirming the failure is specific to the broken tags.

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

func TestContractEventParquetWriterFailsAtSchemaCreation(t *testing.T) {
	// The ContractEventOutputParquet struct has []interface{} fields (Topics,
	// TopicsDecoded) tagged with scalar type=BYTE_ARRAY but no valuetype.
	// parquet-go cannot build a valid schema from these tags, so
	// NewParquetWriter fails before any rows are written.
	//
	// This means `export_contract_events --write-parquet` cannot produce
	// a Parquet file at all — it hits a fatal error at writer creation.

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

	// This is the exact call made by cmd.WriteParquet for contract events:
	//   writer.NewParquetWriter(parquetFile, new(transform.ContractEventOutputParquet), 1)
	pw, err := writer.NewParquetWriter(pf, new(ContractEventOutputParquet), 1)
	if pw != nil {
		pw.WriteStop()
	}

	if err == nil {
		t.Fatalf("expected NewParquetWriter to fail for ContractEventOutputParquet, but it succeeded")
	}

	t.Logf("NewParquetWriter error: %v", err)

	// The error should reference the invalid type mapping for the slice fields
	errStr := err.Error()
	if !strings.Contains(errStr, "not a valid Type") {
		t.Logf("Warning: error message does not contain 'not a valid Type': %s", errStr)
	}

	// Verify that a known-good Parquet schema (TtlOutputParquet, no slices)
	// succeeds, confirming the issue is specific to schemas with badly-tagged
	// slice fields like ContractEventOutputParquet.
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

	t.Log("CONFIRMED: ContractEventOutputParquet schema creation fails while TtlOutputParquet succeeds.")
	t.Log("This means export_contract_events --write-parquet cannot create a Parquet writer at all.")
}
```

### Test Output

```
=== RUN   TestContractEventParquetWriterFailsAtSchemaCreation
    data_integrity_poc_test.go:46: NewParquetWriter error: failed to create schema from tag map: type : not a valid Type string
    data_integrity_poc_test.go:69: CONFIRMED: ContractEventOutputParquet schema creation fails while TtlOutputParquet succeeds.
    data_integrity_poc_test.go:70: This means export_contract_events --write-parquet cannot create a Parquet writer at all.
--- PASS: TestContractEventParquetWriterFailsAtSchemaCreation (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.715s
```
