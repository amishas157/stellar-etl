# H001: Ledger Parquet schema cannot encode evicted-key string arrays

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledgers --write-parquet` should create a valid Parquet file that preserves `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` as repeated string columns, even when those arrays are empty. The Parquet writer should initialize successfully before any rows are written.

## Mechanism

`LedgerOutputParquet` declares `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` as `[]string`, but their Parquet tags only set `type=BYTE_ARRAY, convertedtype=UTF8` and omit the LIST element metadata (`valuetype=...`). parquet-go's slice handling constructs the child element from `GetValueTagMap()`, which copies only `ValueType`; because these tags never set `valuetype`, the child element type is empty and `NewSchemaElementFromTagMap()` fails with an invalid-type error before ledger Parquet export can start.

## Trigger

Run `export_ledgers --write-parquet` on any ledger range without cloud upload so execution reaches `WriteParquet(...)`. Writer initialization for `new(transform.LedgerOutputParquet)` should fail before the first row is emitted, regardless of whether the evicted-key arrays happen to be empty in the current batch.

## Target Code

- `internal/transform/schema_parquet.go:27-28` — `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` are `[]string` tagged without `valuetype`
- `internal/transform/schema_parquet.go:59` — `ExtraSigners` shows the list-tag pattern this codebase uses when a repeated string field is configured correctly
- `cmd/export_ledgers.go:70-72` — ledger export reaches `WriteParquet(..., new(transform.LedgerOutputParquet))`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/schema/schemahandler.go:305-334` — LIST handling builds the element schema from `GetValueTagMap(item.Info)`
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/common/common.go:553-567` — `GetValueTagMap` copies only `ValueType`/`ValueConvertedType` into the child tag
- `/Users/amisha.singla/go/pkg/mod/github.com/xitongsys/parquet-go@v1.6.2/schema/schemahandler.go:388-391` — empty child type causes `failed to create schema from tag map`

## Evidence

The ledger schema's repeated string fields do not follow the tag pattern already used successfully for `TransactionOutputParquet.ExtraSigners`. In parquet-go's reflection path, slice element typing comes from `ValueType`, not from the Go `[]string` element kind, so these two ledger fields look structurally identical to the already-confirmed ContractEvent/ContractData Parquet schema failures.

## Anti-Evidence

If parquet-go inferred the primitive element type directly from `[]string`, the missing `valuetype` would be harmless. The library code cited above does not do that in the LIST branch: it passes an empty child `Type` through to `NewSchemaElementFromTagMap()`, which is why this path still looks viable.
