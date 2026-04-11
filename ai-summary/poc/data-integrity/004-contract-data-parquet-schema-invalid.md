# H004: `ContractDataOutputParquet` cannot create a Parquet writer

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data --write-parquet`
encounters contract-data rows, it should write a valid Parquet file preserving
the same `key`, `key_decoded`, `val`, and `val_decoded` payloads that the JSON
export already emits for those ledger entries.

## Mechanism

`ContractDataOutputParquet` models `key`, `key_decoded`, `val`, and
`val_decoded` as plain `interface{}` fields tagged with `type=MAP,
convertedtype=MAP,...`. `cmd.WriteParquet()` feeds that struct to
`writer.NewParquetWriter(...)`, and a probe against the production schema fails
immediately with `failed to create schema from tag map: type MAP: not a valid
Type string`, so any contract-data Parquet export aborts before the first row
is written.

## Trigger

1. Run `export_ledger_entry_changes --export-contract-data --write-parquet` on
   any batch that contains contract-data ledger entries.
2. Reach `exportTransformedData()`'s `transform.ContractDataOutput` branch.
3. Observe Parquet writer initialization fail for
   `new(transform.ContractDataOutputParquet)`, leaving JSON output only and no
   usable Parquet artifact for the requested contract-data batch.

## Target Code

- `cmd/export_ledger_entry_changes.go:321-347` — selects
  `ContractDataOutputParquet` for contract-data batches
- `cmd/export_ledger_entry_changes.go:370-372` — calls `WriteParquet()` for the
  selected schema
- `cmd/command_utils.go:162-180` — fatals when parquet-go rejects schema
  creation
- `internal/transform/schema_parquet.go:260-281` — declares MAP-typed
  `interface{}` fields for `key`/`val` payloads
- `internal/transform/parquet_converter.go:292-314` — copies the dynamic JSON
  payloads directly into those Parquet fields

## Evidence

An ad-hoc probe using `new(transform.ContractDataOutputParquet)` and the
production parquet-go writer returned `failed to create schema from tag map:
type MAP: not a valid Type string` before any row data was written. The schema
tags ask parquet-go to derive a MAP from opaque `interface{}` fields without a
concrete repeated key/value layout, which is exactly the kind of tag/field
combination the writer rejects.

## Anti-Evidence

JSON export of contract data still works because the dynamic fields are simply
marshaled as JSON. The existing Parquet omission of `ledger_key_hash_base_64`
is a separate bug pattern; even after that omission is addressed, the current
schema definition still prevents any contract-data Parquet writer from being
constructed.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `ContractDataOutputParquet` struct declares four `interface{}` fields (`Key`, `KeyDecoded`, `Val`, `ValDecoded`) with parquet tags `type=MAP, convertedtype=MAP`. When `xitongsys/parquet-go@v1.6.2` processes this struct via reflection, `interface{}` fields have `reflect.Interface` kind — not `reflect.Map` — so the schema handler falls through to the primitive-type parsing path, where `parquet.TypeFromString("MAP")` fails because MAP is not a valid Parquet primitive type. This causes `writer.NewParquetWriter` to return an error, which `WriteParquet` handles by calling `cmdLogger.Fatal`, terminating the entire export job.

### Code Paths Examined

- `internal/transform/schema_parquet.go:261-281` — `ContractDataOutputParquet` struct with four `interface{}` fields tagged `type=MAP`
- `parquet-go/schema/schemahandler.go:300-420` — Schema handler reflects on struct fields; branches on `GoType.Kind()` for Slice/Map but `interface{}` falls through to else clause
- `parquet-go/common/common.go:293-330` — `NewSchemaElementFromTagMap` calls `parquet.TypeFromString(info.Type)` where `info.Type="MAP"`, which returns error
- `parquet-go/parquet/parquet.go:61-82` — `TypeFromString` only accepts primitive types (BOOLEAN, INT32, INT64, etc.); MAP is not valid
- `cmd/command_utils.go:162-180` — `WriteParquet` calls `cmdLogger.Fatal` on schema creation failure
- `cmd/export_ledger_entry_changes.go:344-347` — `ContractDataOutput` case sets `skip=false` and selects `ContractDataOutputParquet` schema
- `cmd/export_ledger_entry_changes.go:370-372` — Calls `WriteParquet` when `!skip && writeParquet`

### Findings

Empirically confirmed: running `schema.NewSchemaHandlerFromStruct(new(ContractDataOutputParquet))` produces error `"failed to create schema from tag map: type MAP: not a valid Type string"`. The root cause is that `parquet-go` dispatches MAP handling based on Go's `reflect.Map` kind, but `interface{}` has `reflect.Interface` kind. The library correctly handles `map[string]string` fields tagged as MAP, but `interface{}` fields never reach that code path. This is the same class of bug as H003 (ContractEventOutputParquet), but affects a different entity type (contract data vs contract events). The `ExtraSigners []string` field at line 59 also has `type=MAP` in its tag but works because its Go type is `[]string` (reflect.Slice), which takes the LIST branch before MAP is evaluated.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create a new `contract_data_parquet_test.go`)
- **Setup**: Import `github.com/xitongsys/parquet-go/schema`
- **Steps**: Call `schema.NewSchemaHandlerFromStruct(new(ContractDataOutputParquet))` and check the returned error
- **Assertion**: Assert that `err` is non-nil and contains `"not a valid Type string"`, demonstrating that the Parquet writer cannot be constructed for `ContractDataOutputParquet`

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestContractDataParquetSchemaInvalid"
**Test Language**: Go

### Demonstration

The test calls `schema.NewSchemaHandlerFromStruct(new(ContractDataOutputParquet))` using the production Parquet struct and confirms that parquet-go returns an error containing "not a valid Type string". This proves that any `--write-parquet` export of contract-data ledger entries will fail at writer initialization, aborting the entire export job via `cmdLogger.Fatal` before a single row is written.

### Test Body

```go
func TestContractDataParquetSchemaInvalid(t *testing.T) {
	_, err := schema.NewSchemaHandlerFromStruct(new(ContractDataOutputParquet))
	if err == nil {
		t.Fatal("expected schema creation to fail for ContractDataOutputParquet, but it succeeded")
	}
	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Fatalf("unexpected error message: %v", err)
	}
	t.Logf("BUG CONFIRMED: ContractDataOutputParquet schema creation failed: %v", err)
	t.Logf("The interface{} fields (Key, KeyDecoded, Val, ValDecoded) tagged with type=MAP cannot be handled by parquet-go")
	t.Logf("Any --write-parquet export of contract-data ledger entries will abort before the first row is written")
}
```

### Test Output

```
=== RUN   TestContractDataParquetSchemaInvalid
    data_integrity_poc_test.go:267: BUG CONFIRMED: ContractDataOutputParquet schema creation failed: failed to create schema from tag map: type MAP: not a valid Type string
    data_integrity_poc_test.go:268: The interface{} fields (Key, KeyDecoded, Val, ValDecoded) tagged with type=MAP cannot be handled by parquet-go
    data_integrity_poc_test.go:269: Any --write-parquet export of contract-data ledger entries will abort before the first row is written
--- PASS: TestContractDataParquetSchemaInvalid (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.746s
```
