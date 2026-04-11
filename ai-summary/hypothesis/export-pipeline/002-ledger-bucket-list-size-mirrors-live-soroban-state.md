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
