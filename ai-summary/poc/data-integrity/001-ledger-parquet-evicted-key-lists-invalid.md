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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the full schema creation path through parquet-go v1.6.2 and confirmed that `LedgerOutputParquet` cannot create a Parquet writer due to the `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` `[]string` fields. These tags specify only `type=BYTE_ARRAY, convertedtype=UTF8` without `valuetype=BYTE_ARRAY`. When parquet-go's schema handler processes a `[]string` field without `repetitiontype=REPEATED`, it enters the LIST branch, creates a child element via `GetValueTagMap()` which copies `src.ValueType` (empty string) into `res.Type`, and then `NewSchemaElementFromTagMap` fails on `TypeFromString("")`. I empirically reproduced the error by running `schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))` in-package.

### Code Paths Examined

- `internal/transform/schema_parquet.go:27-28` — `EvictedLedgerKeysType []string` and `EvictedLedgerKeysHash []string` with tag `type=BYTE_ARRAY, convertedtype=UTF8` but no `valuetype` key
- `internal/transform/schema_parquet.go:59` — `ExtraSigners []string` with correct tag `type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8` — the working pattern
- `internal/transform/schema_parquet.go:66` — `SorobanResourcesArchivedEntries []uint32` with `repetitiontype=REPEATED` — takes a different code path (simple REPEATED, not LIST)
- `parquet-go@v1.6.2/schema/schemahandler.go:297-334` — Slice without REPEATED enters LIST branch; line 324 calls `GetValueTagMap(item.Info)` to derive child element type
- `parquet-go@v1.6.2/common/common.go:553-567` — `GetValueTagMap` sets `res.Type = src.ValueType`; since tag has no `valuetype=...`, this is empty string
- `parquet-go@v1.6.2/common/common.go:293-308` — `NewSchemaElementFromTagMap` calls `TypeFromString("")` which returns error `"not a valid Type string"`
- `parquet-go@v1.6.2/parquet/parquet.go:61-81` — `TypeFromString` switch-cases only match primitive types (BOOLEAN, INT32, INT64, FLOAT, DOUBLE, BYTE_ARRAY, FIXED_LEN_BYTE_ARRAY); empty string falls through to error
- `cmd/export_ledgers.go:70-72` — `WriteParquet(transformedLedgers, parquetPath, new(transform.LedgerOutputParquet))` passes the broken schema
- `cmd/command_utils.go:162-180` — `WriteParquet` calls `cmdLogger.Fatal` on schema error, terminating the process

### Findings

**Empirically confirmed.** Running `schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))` produces:
```
ERROR: failed to create schema from tag map: type : not a valid Type string
```

The root cause is identical to H003 (ContractEventOutputParquet) and H004 (ContractDataOutputParquet) but affects a different entity. The two `[]string` fields on `LedgerOutputParquet` lack the `valuetype=BYTE_ARRAY` tag key that parquet-go requires for LIST element type derivation.

**Severity upgraded to High**: This is **complete data loss** for ledger Parquet exports. `export_ledgers --write-parquet` is entirely non-functional — the command fatals via `cmdLogger.Fatal` before any Parquet row is written. The JSON export path still works (it runs before the Parquet path), but any pipeline relying on Parquet ledger output receives zero data. Note also that at line 71, `MaybeUpload` is called *before* `WriteParquet` at line 72, meaning the upload of a nonexistent/empty Parquet file may occur before the fatal error.

The fix is straightforward — change the tags to match the `ExtraSigners` pattern:
```go
EvictedLedgerKeysType []string `parquet:"name=evicted_ledger_keys_type, type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8"`
EvictedLedgerKeysHash []string `parquet:"name=evicted_ledger_keys_hash, type=MAP, convertedtype=LIST, valuetype=BYTE_ARRAY, valueconvertedtype=UTF8"`
```

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create `internal/transform/ledger_parquet_test.go`)
- **Setup**: Import `github.com/xitongsys/parquet-go/schema`
- **Steps**: Call `schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))` and check the returned error
- **Assertion**: Assert that `err` is non-nil and contains `"not a valid Type string"`, demonstrating that the Parquet writer cannot be constructed for `LedgerOutputParquet`

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestLedgerParquetEvictedKeyListsInvalid"
**Test Language**: Go

### Demonstration

The test calls `schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))` and confirms that the Parquet schema handler cannot be created due to the missing `valuetype` tag on the `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` `[]string` fields. The error message "not a valid Type string" proves that `export_ledgers --write-parquet` will fatal before writing any Parquet data, causing complete data loss for ledger Parquet exports.

### Test Body

```go
package transform

import (
	"strings"
	"testing"

	"github.com/xitongsys/parquet-go/schema"
)

func TestLedgerParquetEvictedKeyListsInvalid(t *testing.T) {
	// Attempt to create a Parquet schema handler from LedgerOutputParquet.
	// The EvictedLedgerKeysType and EvictedLedgerKeysHash fields are []string
	// with tags that lack valuetype=BYTE_ARRAY, so parquet-go cannot derive
	// the LIST child element type and should fail.
	_, err := schema.NewSchemaHandlerFromStruct(new(LedgerOutputParquet))
	if err == nil {
		t.Fatal("expected schema creation to fail for LedgerOutputParquet due to missing valuetype on []string fields, but it succeeded")
	}
	if !strings.Contains(err.Error(), "not a valid Type string") {
		t.Errorf("unexpected error message: %v", err)
	}
	t.Logf("Schema creation failed as expected: %v", err)
}
```

### Test Output

```
=== RUN   TestLedgerParquetEvictedKeyListsInvalid
    data_integrity_poc_test.go:22: Schema creation failed as expected: failed to create schema from tag map: type : not a valid Type string
--- PASS: TestLedgerParquetEvictedKeyListsInvalid (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.874s
```
