# H003: `ContractEventOutputParquet` cannot create a Parquet writer

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_contract_events --write-parquet` should serialize the same contract
event payload already present in JSON — including `topics`, `topics_decoded`,
`data`, and `data_decoded` — into a valid Parquet file. Enabling Parquet should
not stop the export before any rows are written.

## Mechanism

`ContractEventOutputParquet` declares `topics`/`topics_decoded` as UTF-8
`BYTE_ARRAY` columns but stores them as `[]interface{}`, and declares
`data`/`data_decoded` as scalar UTF-8 `BYTE_ARRAY` columns backed by
`interface{}`. `cmd.WriteParquet()` passes that malformed schema directly to
`writer.NewParquetWriter(...)`, and a probe against the production struct fails
immediately with `failed to create schema from tag map: type : not a valid Type
string`, so Parquet export aborts before any contract-event row can be written.

## Trigger

1. Run `export_contract_events --write-parquet` for any ledger range.
2. Reach the Parquet branch that calls `WriteParquet(..., new(transform.ContractEventOutputParquet))`.
3. Observe writer initialization fail before the first row is emitted, leaving
   no valid Parquet artifact for the requested contract-event export.

## Target Code

- `cmd/export_contract_events.go:63-65` — selects `ContractEventOutputParquet`
  for the Parquet export path
- `cmd/command_utils.go:162-180` — `WriteParquet()` calls
  `writer.NewParquetWriter(schema, ...)` and fatals on schema creation errors
- `internal/transform/schema_parquet.go:382-399` — declares
  `topics`/`topics_decoded`/`data`/`data_decoded` with incompatible
  `interface{}`-typed fields
- `internal/transform/parquet_converter.go:425-441` — copies raw interface
  payloads directly into the malformed Parquet row type
- `internal/transform/schema.go:640-657` — JSON schema shows the source fields
  really are dynamic interface payloads that need explicit Parquet encoding

## Evidence

An ad-hoc probe using the repository's production type
`new(transform.ContractEventOutputParquet)` and the same parquet-go writer path
returned `failed to create schema from tag map: type : not a valid Type
string` before any row write. The failure is consistent with the current schema:
there is no valid repeated/list annotation for `[]interface{}` fields, and the
scalar `interface{}` fields likewise do not tell parquet-go how to derive a
concrete physical type.

## Anti-Evidence

The JSON path is unaffected because `ExportEntry()` simply marshals the dynamic
payload with `json.Marshal`. If parquet-go had custom support for these exact
`interface{}` combinations the schema might be salvageable, but the current
writer initialization rejects it outright.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced the full schema creation path through parquet-go v1.6.2 and confirmed that `ContractEventOutputParquet` cannot create a Parquet writer. The `[]interface{}` fields (`Topics`, `TopicsDecoded`) trigger the LIST handling path in `schema.NewSchemaHandlerFromStruct`, which extracts `ValueType` from the tag via `GetValueTagMap()`. Since the tags specify only `type=BYTE_ARRAY` (not `valuetype=BYTE_ARRAY`), the child element's type is an empty string, causing `parquet.TypeFromString("")` to fail. I ran an empirical confirmation using `go run` against the exact production struct and reproduced the error: `failed to create schema from tag map: type : not a valid Type string`.

### Code Paths Examined

- `internal/transform/schema_parquet.go:382-399` — `ContractEventOutputParquet` struct declares `Topics []interface{}` and `TopicsDecoded []interface{}` with tag `type=BYTE_ARRAY` but no `valuetype` key; also declares `Data interface{}` and `DataDecoded interface{}` as scalar interface types
- `cmd/export_contract_events.go:63-64` — passes `new(transform.ContractEventOutputParquet)` to `WriteParquet`
- `cmd/command_utils.go:162-180` — `WriteParquet()` calls `writer.NewParquetWriter(parquetFile, schema, 1)` and fatals on error
- `parquet-go@v1.6.2/writer/writer.go:56-100` — `NewParquetWriter` calls `schema.NewSchemaHandlerFromStruct(obj)` at line 92
- `parquet-go@v1.6.2/schema/schemahandler.go:297-334` — slice handling for `[]interface{}`: enters LIST branch, calls `GetValueTagMap(item.Info)` at line 324 which returns Tag with empty `Type` (since `ValueType` was never set), pushes child element with `GoType = interface{}`
- `parquet-go@v1.6.2/common/common.go:553-568` — `GetValueTagMap` copies `src.ValueType` to `res.Type`; since tag has no `valuetype=...` key, this is empty string
- `parquet-go@v1.6.2/schema/schemahandler.go:388-391` — child element falls to the `else` branch, `NewSchemaElementFromTagMap` fails at `parquet.TypeFromString("")`
- `parquet-go@v1.6.2/common/common.go:293-308` — `NewSchemaElementFromTagMap` returns error: `"type : not a valid Type string"`
- `internal/transform/parquet_converter.go:425-441` — `ToParquet()` copies raw `interface{}` values directly, which would also fail at write time even if schema succeeded

### Findings

**The bug is confirmed and empirically reproduced.** Running `schema.NewSchemaHandlerFromStruct(new(ContractEventOutputParquet))` produces:
```
ERROR: failed to create schema from tag map: type : not a valid Type string
```

The root cause is a mismatch between Go field types and parquet-go's tag-based schema derivation:

1. **`[]interface{}` fields** (Topics, TopicsDecoded): parquet-go's LIST handling extracts the element type from the `valuetype` tag key, not `type`. Since the tags specify only `type=BYTE_ARRAY`, the element type resolves to empty string, failing validation.

2. **`interface{}` fields** (Data, DataDecoded): While the scalar `interface{}` fields technically pass schema creation (the `type=BYTE_ARRAY` tag satisfies `NewSchemaElementFromTagMap`), they would fail at write time because parquet-go's marshal path cannot reflect into `interface{}` to extract bytes.

**Severity upgraded to High**: This is not merely an operational issue — it is a **complete data loss** for contract event Parquet exports. The `--write-parquet` flag on `export_contract_events` is entirely non-functional. The command fatals via `cmdLogger.Fatal` at line 171, which means the export aborts after the JSON file is already written and potentially uploaded. Any pipeline relying on Parquet contract event output receives zero data.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or create `internal/transform/contract_event_parquet_test.go`)
- **Setup**: Import `schema "github.com/xitongsys/parquet-go/schema"` and `writer "github.com/xitongsys/parquet-go/writer"` along with `local "github.com/xitongsys/parquet-go-source/local"`
- **Steps**:
  1. Call `schema.NewSchemaHandlerFromStruct(new(transform.ContractEventOutputParquet))` and assert error is non-nil
  2. Alternatively, create a local parquet file writer and call `writer.NewParquetWriter(pf, new(transform.ContractEventOutputParquet), 1)` and assert error is non-nil
  3. Verify the error message contains "not a valid Type string"
- **Assertion**: `require.Error(t, err)` and `require.Contains(t, err.Error(), "not a valid Type string")` — the schema cannot be created, so no Parquet file can be produced for contract events
