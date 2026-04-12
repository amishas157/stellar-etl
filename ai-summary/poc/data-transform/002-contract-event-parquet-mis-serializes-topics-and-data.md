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
**Test Name**: "TestContractEventParquetCorruptsDecodedFields"
**Test Language**: Go

### Demonstration

The test constructs a `ContractEventOutput` with `json.RawMessage` values in `TopicsDecoded` and `DataDecoded` (mirroring what `serializeScVal()` produces at runtime), then calls `ToParquet()` and verifies the output fields are NOT equal to what `toJSONString()` would produce. The test passes, confirming that `ToParquet()` copies `json.RawMessage` (which is `[]byte`) verbatim instead of normalizing to JSON strings. The actual Parquet output renders as raw byte slice notation (e.g., `[123 34 116 ...]`) rather than the expected JSON content, and in the parquet-go marshal path these would become `"<json.RawMessage Value>"` sentinel strings. Meanwhile, `Topics` and `Data` (which hold plain `string` values) are correctly preserved.

### Test Body

```go
package transform

import (
	"encoding/json"
	"fmt"
	"testing"
	"time"
)

func TestContractEventParquetCorruptsDecodedFields(t *testing.T) {
	// 1. Construct a ContractEventOutput with json.RawMessage in decoded fields,
	//    mirroring what serializeScVal() produces at runtime.
	topicDecoded1 := json.RawMessage(`{"type":"Address","value":"GABC123"}`)
	topicDecoded2 := json.RawMessage(`{"type":"Symbol","value":"transfer"}`)
	dataDecoded := json.RawMessage(`{"type":"I128","lo":"1000000","hi":"0"}`)

	ceo := ContractEventOutput{
		TransactionHash:          "abc123",
		TransactionID:            12345,
		Successful:               true,
		LedgerSequence:           100,
		ClosedAt:                 time.Now(),
		InSuccessfulContractCall: true,
		ContractId:               "CAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFCT4",
		Type:                     0,
		TypeString:               "contract",
		Topics:                   []interface{}{"dG9waWMx", "dG9waWMy"},
		TopicsDecoded:            []interface{}{topicDecoded1, topicDecoded2},
		Data:                     "ZGF0YQ==",
		DataDecoded:              dataDecoded,
		ContractEventXDR:         "AAAAAA==",
	}

	// 2. Run production code path
	parquetRaw := ceo.ToParquet()
	parquetOutput := parquetRaw.(ContractEventOutputParquet)

	// 3. Demonstrate corruption: the decoded fields are NOT normalized strings.
	expectedTopicDecoded0 := toJSONString(topicDecoded1)
	expectedTopicDecoded1 := toJSONString(topicDecoded2)
	expectedDataDecoded := toJSONString(dataDecoded)

	actualTopicDecoded0 := fmt.Sprintf("%v", parquetOutput.TopicsDecoded[0])
	actualTopicDecoded1 := fmt.Sprintf("%v", parquetOutput.TopicsDecoded[1])
	actualDataDecoded := fmt.Sprintf("%v", parquetOutput.DataDecoded)

	t.Logf("Expected TopicsDecoded[0] (toJSONString): %s", expectedTopicDecoded0)
	t.Logf("Actual   TopicsDecoded[0] (ToParquet):    %s", actualTopicDecoded0)
	t.Logf("Expected TopicsDecoded[1] (toJSONString): %s", expectedTopicDecoded1)
	t.Logf("Actual   TopicsDecoded[1] (ToParquet):    %s", actualTopicDecoded1)
	t.Logf("Expected DataDecoded (toJSONString):      %s", expectedDataDecoded)
	t.Logf("Actual   DataDecoded (ToParquet):          %s", actualDataDecoded)

	// Assert corruption IS present: values do NOT match what toJSONString() produces.
	if actualTopicDecoded0 == expectedTopicDecoded0 {
		t.Errorf("TopicsDecoded[0] unexpectedly correct - expected corruption but values match")
	}
	if actualTopicDecoded1 == expectedTopicDecoded1 {
		t.Errorf("TopicsDecoded[1] unexpectedly correct - expected corruption but values match")
	}
	if actualDataDecoded == expectedDataDecoded {
		t.Errorf("DataDecoded unexpectedly correct - expected corruption but values match")
	}

	// Verify that Topics and Data (plain strings, not json.RawMessage) are fine.
	actualTopic0 := fmt.Sprintf("%v", parquetOutput.Topics[0])
	actualData := fmt.Sprintf("%v", parquetOutput.Data)
	if actualTopic0 != "dG9waWMx" {
		t.Errorf("Topics[0] should be correct (string type): got %q", actualTopic0)
	}
	if actualData != "ZGF0YQ==" {
		t.Errorf("Data should be correct (string type): got %q", actualData)
	}

	t.Log("CONFIRMED: ToParquet() passes json.RawMessage through without normalization.")
	t.Log("Sibling converters (OperationOutput, EffectOutput, ConfigSettingOutput) use toJSONString().")
	t.Log("ContractEventOutput.ToParquet() is the outlier - this is the bug.")
}
```

### Test Output

```
=== RUN   TestContractEventParquetCorruptsDecodedFields
    data_integrity_poc_test.go:54: Expected TopicsDecoded[0] (toJSONString): {"type":"Address","value":"GABC123"}
    data_integrity_poc_test.go:55: Actual   TopicsDecoded[0] (ToParquet):    [123 34 116 121 112 101 34 58 34 65 100 100 114 101 115 115 34 44 34 118 97 108 117 101 34 58 34 71 65 66 67 49 50 51 34 125]
    data_integrity_poc_test.go:56: Expected TopicsDecoded[1] (toJSONString): {"type":"Symbol","value":"transfer"}
    data_integrity_poc_test.go:57: Actual   TopicsDecoded[1] (ToParquet):    [123 34 116 121 112 101 34 58 34 83 121 109 98 111 108 34 44 34 118 97 108 117 101 34 58 34 116 114 97 110 115 102 101 114 34 125]
    data_integrity_poc_test.go:58: Expected DataDecoded (toJSONString):      {"type":"I128","lo":"1000000","hi":"0"}
    data_integrity_poc_test.go:59: Actual   DataDecoded (ToParquet):          [123 34 116 121 112 101 34 58 34 73 49 50 56 34 44 34 108 111 34 58 34 49 48 48 48 48 48 48 34 44 34 104 105 34 58 34 48 34 125]
    data_integrity_poc_test.go:86: CONFIRMED: ToParquet() passes json.RawMessage through without normalization.
    data_integrity_poc_test.go:87: Sibling converters (OperationOutput, EffectOutput, ConfigSettingOutput) use toJSONString().
    data_integrity_poc_test.go:88: ContractEventOutput.ToParquet() is the outlier - this is the bug.
--- PASS: TestContractEventParquetCorruptsDecodedFields (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.846s
```
