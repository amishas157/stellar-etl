# H002: `total_byte_size_of_bucket_list` mirrors live Soroban state size

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Wrong ledger size field mapping
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.total_byte_size_of_bucket_list` should contain a bucket-list size metric if one exists in the ledger-close source data. If the current close-meta versions do not expose that metric, the exporter should leave the field zero-valued or otherwise clearly unsupported rather than filling it with a different ledger size.

## Mechanism

`TransformLedger()` only reads `TotalByteSizeOfLiveSorobanState` from `LedgerCloseMetaV1/V2`, stores it in `outputTotalByteSizeOfLiveSorobanState`, and then assigns that same value to both `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState`. The current XDR definitions for `LedgerCloseMetaV0/V1/V2` expose `TotalByteSizeOfLiveSorobanState` but no bucket-list byte-size field, so the exported bucket-list column is fabricated from an unrelated metric on every affected ledger row.

## Trigger

Run `export_ledgers` on any ledger whose close meta is V1 or V2 and whose `TotalByteSizeOfLiveSorobanState` is non-zero. The JSON and Parquet rows will emit identical values for `total_byte_size_of_bucket_list` and `total_byte_size_of_live_soroban_state`.

## Target Code

- `internal/transform/ledger.go:61-86` — the transformer only reads `TotalByteSizeOfLiveSorobanState`
- `internal/transform/ledger.go:104-129` — both output columns are filled from `outputTotalByteSizeOfLiveSorobanState`
- `internal/transform/schema.go:34-35` — the schema models `total_byte_size_of_bucket_list` and `total_byte_size_of_live_soroban_state` as distinct columns
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:19600-19609` — `LedgerCloseMetaV1` exposes `TotalByteSizeOfLiveSorobanState` but no bucket-list size field
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:19855-19864` — `LedgerCloseMetaV2` exposes `TotalByteSizeOfLiveSorobanState` but no bucket-list size field

## Evidence

The internal transformer never reads a second source metric before populating `TotalByteSizeOfBucketList`. The generated XDR structs confirm that the current close-meta versions only provide live Soroban state size, so the exporter is not preserving a real bucket-list value when it writes the duplicate column.

## Anti-Evidence

The schema may retain `total_byte_size_of_bucket_list` for backward compatibility with downstream tables. But unlike Protocol 23 rename aliases, the current XDR model does not expose a same-semantic bucket-list field that could justify copying the live-state value into this column.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of ai-summary/fail/utilities/012-ledger-bucket-size-column-is-a-legacy-alias.md
**Failed At**: reviewer

### Trace Summary

This hypothesis is substantially equivalent to a previously investigated finding. `ai-summary/fail/utilities/012-ledger-bucket-size-column-is-a-legacy-alias.md` examined the identical code path (`TransformLedger()` assigning both `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState` from the same variable) and concluded NOT_VIABLE. Additionally, `ai-summary/fail/export-pipeline/004-config-setting-bucket-columns-mirror-live-state.md` investigated the same conceptual pattern for config-setting exports and found it to be intentional Protocol 23 backward-compatibility aliasing.

### Code Paths Examined

- `internal/transform/ledger.go:61-91` — Reads `TotalByteSizeOfLiveSorobanState` from `lcmV1` and `lcmV2`, stores in single local variable
- `internal/transform/ledger.go:125-126` — Both output fields assigned from `outputTotalByteSizeOfLiveSorobanState`
- `internal/transform/schema.go:34-35` — Two distinct columns in the output schema

### Why It Failed

Duplicate of a prior investigation. The earlier review (`012-ledger-bucket-size-column-is-a-legacy-alias.md`) confirmed that the current XDR `LedgerCloseMeta` versions do not expose a distinct bucket-list byte-size field. The `TotalByteSizeOfBucketList` column is a legacy compatibility alias from the Protocol 23 rename (`TotalByteSizeOfBucketList` → `TotalByteSizeOfLiveSorobanState`), following the same dual-column aliasing pattern documented in `004-config-setting-bucket-columns-mirror-live-state.md`. Without a distinct XDR source field, there is no wrong-source mapping — the code is working as designed.

### Lesson Learned

Ledger-level `bucket_list` columns follow the same Protocol 23 backward-compatibility aliasing pattern as config-setting `bucket_list` columns. Both retain old column names populated from renamed XDR fields.
