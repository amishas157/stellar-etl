# H004b: Contract-data parquet conversion drops `ledger_key_hash_base_64`

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Non-financial field contains wrong data (latent — blocked by upstream writer-init failure)
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `ContractDataOutput.ToParquet()` converts a contract-data record for parquet serialization, the `LedgerKeyHashBase64` field should be preserved in the resulting `ContractDataOutputParquet` struct, just as it is preserved in the JSON output path.

## Mechanism

`TransformContractData()` computes `LedgerKeyHashBase64`, and the corresponding JSON tests assert it. But the parquet struct `ContractDataOutputParquet` and its `ToParquet()` converter have no destination for that field. The value is silently discarded during the conversion.

**Important caveat**: In the live export pipeline, `ContractDataOutputParquet` uses MAP-typed parquet tags (`Key`, `KeyDecoded`, `Val`, `ValDecoded`) that cause `writer.NewParquetWriter()` to fail with `"type MAP: not a valid Type string"`. This means the export pipeline crashes before writing any contract_data parquet rows. The `LedgerKeyHashBase64` omission is therefore a **latent bug** — it would manifest if the MAP-type writer-init issue were fixed first.

## Trigger

Call `ContractDataOutput.ToParquet()` on any record where `LedgerKeyHashBase64` is populated. The returned `ContractDataOutputParquet` struct has no field to hold the value.

## Target Code

- `internal/transform/contract_data.go:TransformContractData:65-78,135-156` — computes and emits `LedgerKeyHashBase64`
- `internal/transform/schema.go:ContractDataOutput:516-537` — JSON schema includes `ledger_key_hash_base_64`
- `internal/transform/contract_data_test.go:142-164` — expected contract-data output includes `LedgerKeyHashBase64`
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:261-281` — parquet schema omits `LedgerKeyHashBase64`
- `internal/transform/parquet_converter.go:ContractDataOutput.ToParquet:292-313` — no mapping for the field
- `internal/transform/schema_parquet.go:ContractDataOutputParquet:276-279` — MAP-type tags that block writer init

## Evidence

`ledger_key_hash_base_64` is not dead JSON-only baggage: the contract-data transform actively computes it from the live ledger key, and the test fixtures treat it as part of the expected export shape. The `ToParquet()` converter silently drops it.

## Anti-Evidence

The field omission is currently masked by a larger issue: `ContractDataOutputParquet` cannot be used to initialize a parquet writer at all (MAP-type tags are invalid). If the MAP issue is fixed without also adding `LedgerKeyHashBase64`, this bug will become live.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium (latent — blocked by writer-init failure)
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — split from combined finding H004

### Trace Summary

Traced the `ToParquet()` conversion and confirmed the field is missing from the destination struct. Also confirmed that `writer.NewParquetWriter(pf, new(ContractDataOutputParquet), 1)` fails with `"type MAP: not a valid Type string"` due to the `Key`/`KeyDecoded`/`Val`/`ValDecoded` MAP-typed fields. This means the export pipeline never writes contract_data parquet rows, making the `LedgerKeyHashBase64` omission a latent bug at the transform layer.

### Code Paths Examined

- `internal/transform/contract_data.go:65-78` — computes `ledgerKeyHashBase64` from `xdr.MarshalBase64(ledgerKey)`, assigns to output struct at line 155
- `internal/transform/schema.go:536` — `ContractDataOutput.LedgerKeyHashBase64` field present with JSON tag `ledger_key_hash_base_64`
- `internal/transform/schema_parquet.go:261-281` — `ContractDataOutputParquet` has 20 fields; `LedgerKeyHashBase64` is absent
- `internal/transform/schema_parquet.go:276-279` — MAP-type tags on `Key`, `KeyDecoded`, `Val`, `ValDecoded` that break writer init
- `internal/transform/parquet_converter.go:292-313` — `ContractDataOutput.ToParquet()` maps all fields except `LedgerKeyHashBase64`
- `internal/transform/contract_data_test.go:163` — test expects `LedgerKeyHashBase64` populated with a real base64 value

### Findings

The `LedgerKeyHashBase64` field omission is confirmed at the transform layer. However, the export pipeline cannot currently write contract_data parquet rows due to a separate MAP-type schema issue. This bug is latent — scoped to the transform/parquet conversion layer.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go`
- **Setup**: Create a `ContractDataOutput` with non-empty `LedgerKeyHashBase64`
- **Steps**: Call `.ToParquet()`, verify via reflection that the parquet struct lacks the field; also verify the parquet writer init fails for `ContractDataOutputParquet`
- **Assertion**: The parquet struct has no `LedgerKeyHashBase64` field (transform omission), AND the writer init fails (confirming latent status)

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractDataParquetDropsLedgerKeyHashBase64"
**Test Language**: Go

### Demonstration

The test confirms that `LedgerKeyHashBase64` exists on `ContractDataOutput` (JSON schema) but is completely absent from `ContractDataOutputParquet`. When `ToParquet()` is called, the base64-encoded ledger key is silently discarded. The test also verifies that the parquet writer fails to initialize for `ContractDataOutputParquet` with `"type MAP: not a valid Type string"`, confirming this field omission is a latent bug behind a larger initialization failure.

### Test Body

```go
package transform

import (
	"os"
	"reflect"
	"testing"
	"time"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

// TestContractDataParquetDropsLedgerKeyHashBase64 demonstrates that
// ContractDataOutput.ToParquet() drops the LedgerKeyHashBase64 field because
// ContractDataOutputParquet has no corresponding field.
// NOTE: This is scoped to the transform/parquet conversion layer because the
// contract_data parquet writer fails to initialize (MAP type tags), so in the
// live export pipeline this code path currently crashes before writing any rows.
// A PASSING test confirms the transform-layer omission.
func TestContractDataParquetDropsLedgerKeyHashBase64(t *testing.T) {
	// 1. Construct input with a non-empty LedgerKeyHashBase64
	input := ContractDataOutput{
		ContractId:          "CABCDEFGHIJKLMNOPQRSTUVWXYZ234567ABCDEFGHIJKLMNOPQRSTUV",
		LedgerKeyHash:       "abc123hash",
		LedgerKeyHashBase64: "AAAABgAAAAEAAAAAAAAAAgAAAA==",
		LedgerSequence:      100,
		ClosedAt:            time.Now(),
	}

	if input.LedgerKeyHashBase64 == "" {
		t.Fatal("precondition: LedgerKeyHashBase64 must be set on input")
	}

	// 2. Run the production ToParquet() converter
	parquetResult := input.ToParquet()

	// 3. Assert the parquet struct is missing the LedgerKeyHashBase64 field
	parquetType := reflect.TypeOf(parquetResult)
	if fieldExists(parquetType, "LedgerKeyHashBase64") {
		t.Fatal("Expected LedgerKeyHashBase64 to be MISSING from ContractDataOutputParquet, but it was found — bug may be fixed")
	}

	// 4. Confirm: JSON struct has the field, parquet struct does not
	jsonType := reflect.TypeOf(input)
	if !fieldExists(jsonType, "LedgerKeyHashBase64") {
		t.Fatal("Expected LedgerKeyHashBase64 to exist on ContractDataOutput JSON struct")
	}

	// 5. Verify the contract_data parquet writer FAILS to initialize due to MAP type tags.
	// This means the export pipeline crashes before it can even write rows — the field
	// omission is a latent bug behind a larger initialization failure.
	tmpFile, err := os.CreateTemp("", "contract_data_parquet_*.parquet")
	if err != nil {
		t.Fatal("failed to create temp file:", err)
	}
	tmpPath := tmpFile.Name()
	tmpFile.Close()
	defer os.Remove(tmpPath)

	pf, err := local.NewLocalFileWriter(tmpPath)
	if err != nil {
		t.Fatal("failed to create parquet file writer:", err)
	}
	_, writerErr := writer.NewParquetWriter(pf, new(ContractDataOutputParquet), 1)
	pf.Close()

	if writerErr == nil {
		t.Log("WARNING: parquet writer init succeeded for ContractDataOutputParquet — MAP type handling may have been fixed")
	} else {
		t.Logf("CONFIRMED: ContractDataOutputParquet parquet writer init fails: %v", writerErr)
		t.Log("The LedgerKeyHashBase64 omission is a latent bug behind this initialization failure.")
	}

	t.Logf("BUG CONFIRMED (transform layer): ContractDataOutput.LedgerKeyHashBase64=%q is present in JSON schema "+
		"but ContractDataOutputParquet has no such field — value is dropped by ToParquet()",
		input.LedgerKeyHashBase64)
}

func fieldExists(t reflect.Type, name string) bool {
	for i := 0; i < t.NumField(); i++ {
		if t.Field(i).Name == name {
			return true
		}
	}
	return false
}
```

### Test Output

```
=== RUN   TestContractDataParquetDropsLedgerKeyHashBase64
    data_integrity_poc_test.go:133: CONFIRMED: ContractDataOutputParquet parquet writer init fails: failed to create schema from tag map: type MAP: not a valid Type string
    data_integrity_poc_test.go:134: The LedgerKeyHashBase64 omission is a latent bug behind this initialization failure.
    data_integrity_poc_test.go:137: BUG CONFIRMED (transform layer): ContractDataOutput.LedgerKeyHashBase64="AAAABgAAAAEAAAAAAAAAAgAAAA==" is present in JSON schema but ContractDataOutputParquet has no such field — value is dropped by ToParquet()
--- PASS: TestContractDataParquetDropsLedgerKeyHashBase64 (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.715s
```
