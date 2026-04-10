# H001: Ledger bucket-list byte size mirrors live Soroban state

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_ledgers.total_byte_size_of_bucket_list` should preserve the bucket-list byte size when ledger-close metadata exposes it, and it should never silently mirror `total_byte_size_of_live_soroban_state`. The two exported columns represent different storage measurements and should diverge whenever the underlying values differ.

## Mechanism

`TransformLedger()` reads `TotalByteSizeOfLiveSorobanState` into a local variable named `totalByteSizeOfBucketList` for both `LedgerCloseMetaV1` and `LedgerCloseMetaV2`, then writes that single value into both `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState`. Any ledger with non-zero live Soroban state therefore exports a plausible-but-wrong bucket-list size that just duplicates the live-state measurement.

## Trigger

Run `export_ledgers` on any Soroban-era ledger range where `LedgerCloseMeta` reports a non-zero live Soroban state size. Compare the JSON or Parquet row: `total_byte_size_of_bucket_list` will equal `total_byte_size_of_live_soroban_state` even though the exporter claims they are separate columns.

## Target Code

- `internal/transform/ledger.go:TransformLedger:61-87` — reads `TotalByteSizeOfLiveSorobanState` into a bucket-list local for both LCM versions
- `internal/transform/ledger.go:TransformLedger:104-129` — writes the same value into both ledger output fields
- `internal/transform/schema.go:LedgerOutput:12-38` — schema advertises distinct `total_byte_size_of_bucket_list` and `total_byte_size_of_live_soroban_state` columns

## Evidence

`TransformLedger()` declares only `outputTotalByteSizeOfLiveSorobanState`, not a separate bucket-list accumulator. The V1 path assigns `lcmV1.TotalByteSizeOfLiveSorobanState` at lines 71-72, the V2 path assigns `lcmV2.TotalByteSizeOfLiveSorobanState` at lines 85-86, and the final struct write at lines 125-126 copies that one variable into both JSON fields.

## Anti-Evidence

The current SDK `LedgerCloseMetaV1/V2` types emphasize `TotalByteSizeOfLiveSorobanState`, so the historical bucket-list source may have moved or been removed upstream. But the ETL still exports both columns today, which makes silently duplicating the live-state value into the bucket-list column look like data corruption rather than an intentional omission.
