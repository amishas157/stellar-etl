# H001: Ledger bucket-list size column copies live Soroban state size

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformLedger()` should export `total_byte_size_of_bucket_list` from the `LedgerCloseMetaV1/V2.TotalByteSizeOfBucketList` XDR field and `total_byte_size_of_live_soroban_state` from the separate `TotalByteSizeOfLiveSorobanState` field. When the two upstream counters differ, the JSON and Parquet rows should preserve both values exactly instead of duplicating one of them.

## Mechanism

`internal/transform/ledger.go` reads `TotalByteSizeOfLiveSorobanState` into a local variable named like the bucket-list counter, then writes that same value into both output columns. Because `LedgerOutput` and `LedgerOutputParquet` expose both columns independently, any ledger where bucket-list size and live Soroban state size diverge will silently export a plausible-but-wrong `total_byte_size_of_bucket_list`.

## Trigger

1. Process a ledger-close meta V1 or V2 record whose `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState` are not equal.
2. Run `export_ledgers`.
3. Compare the two exported size columns against the source `LedgerCloseMeta`: `total_byte_size_of_bucket_list` will match the live-state counter instead of the bucket-list counter.

## Target Code

- `internal/transform/ledger.go:61-87` — reads only `TotalByteSizeOfLiveSorobanState` from both V1 and V2 metadata
- `internal/transform/ledger.go:104-129` — assigns the same local value to both `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState`
- `internal/transform/schema.go:12-38` — JSON schema exposes the two counters as distinct columns
- `internal/transform/parquet_converter.go:27-56` — Parquet conversion faithfully propagates the already-corrupted JSON values

## Evidence

The upstream XDR type defines `LedgerCloseMetaV1.TotalByteSizeOfBucketList` as its own `Uint64` field, so there is separate source data available for the bucket-list metric. In `TransformLedger()`, both V1 and V2 branches instead read `TotalByteSizeOfLiveSorobanState`, and the final `LedgerOutput` writes that one value into both exported columns. The ledger unit test's expected output omits both size fields, so the copy-paste mapping is currently untested.

## Anti-Evidence

If the network happens to produce equal values for both counters, the bug is masked because the duplicated export still looks internally consistent. The code also correctly preserves both columns once they are present on `LedgerOutput`, so the issue is isolated to source-field selection inside `TransformLedger()`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of utilities/012 and export-pipeline/018
**Failed At**: reviewer

### Trace Summary

Traced the XDR type definitions in the go-stellar-sdk module across multiple versions. The generated XDR (`xdr_generated.go`) for both `LedgerCloseMetaV1` and `LedgerCloseMetaV2` contains only `TotalByteSizeOfLiveSorobanState` — there is no separate `TotalByteSizeOfBucketList` field. The original field was renamed during Protocol 23. The `total_byte_size_of_bucket_list` export column is a backward-compatible alias intentionally populated from the same single XDR source field.

### Code Paths Examined

- `internal/transform/ledger.go:61-91` — V1 and V2 branches both read `lcmV1.TotalByteSizeOfLiveSorobanState` / `lcmV2.TotalByteSizeOfLiveSorobanState` (the only field available)
- `internal/transform/ledger.go:125-126` — Both output columns assigned from the same variable (intentional aliasing)
- `go-stellar-sdk/xdr/xdr_generated.go:19607,19862` — `LedgerCloseMetaV1` and `LedgerCloseMetaV2` structs only define `TotalByteSizeOfLiveSorobanState`; no `TotalByteSizeOfBucketList` field exists

### Why It Failed

The hypothesis's core premise is false: the current XDR does NOT define a separate `TotalByteSizeOfBucketList` field on `LedgerCloseMetaV1` or `LedgerCloseMetaV2`. The field was renamed to `TotalByteSizeOfLiveSorobanState` as part of Protocol 23. The export schema retains both column names for downstream BigQuery compatibility, intentionally populating both from the single XDR source. This is the same backward-compatibility aliasing pattern already documented in utilities/012 and export-pipeline/018.

### Lesson Learned

This exact hypothesis has now been rejected three times across different subsystems (utilities/012, export-pipeline/018, data-integrity/002). Always verify the current generated XDR struct definitions before assuming two export columns must have two distinct XDR sources. Protocol-level field renames create intentional aliasing in the export schema.
