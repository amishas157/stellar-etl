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
