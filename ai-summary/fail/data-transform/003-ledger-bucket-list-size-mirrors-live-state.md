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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `TransformLedger()` in `internal/transform/ledger.go` lines 61–131. Confirmed that both `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState` output fields receive the same value from `outputTotalByteSizeOfLiveSorobanState`. Then inspected the upstream XDR types `LedgerCloseMetaV1` and `LedgerCloseMetaV2` in `go-stellar-sdk/xdr/Stellar-ledger.x` and the generated Go types. Found that **neither XDR struct has a `totalByteSizeOfBucketList` field** — only `totalByteSizeOfLiveSorobanState` exists. The XDR comment describes it as "Size in bytes of live Soroban state, to support downstream systems calculating storage fees correctly." There is no separate bucket-list-size field anywhere in the Stellar protocol metadata for these LCM versions.

### Code Paths Examined

- `internal/transform/ledger.go:TransformLedger:61-91` — V1/V2 branches both read `TotalByteSizeOfLiveSorobanState` (the only field available) into a misleadingly-named local `totalByteSizeOfBucketList`
- `internal/transform/ledger.go:TransformLedger:125-126` — both output columns assigned from the single variable `outputTotalByteSizeOfLiveSorobanState`
- `go-stellar-sdk/xdr/Stellar-ledger.x` (XDR definition) — `LedgerCloseMetaV1` and `LedgerCloseMetaV2` each define exactly one field: `uint64 totalByteSizeOfLiveSorobanState`. No `totalByteSizeOfBucketList` field exists in the protocol.
- `go-stellar-sdk/xdr/xdr_generated.go:19607,19862` — generated Go types confirm only `TotalByteSizeOfLiveSorobanState Uint64` on both V1 and V2 structs

### Why It Failed

The hypothesis assumes that the XDR exposes two distinct measurements — a bucket-list size and a live-Soroban-state size — and that the code incorrectly reads only one. In reality, the Stellar protocol (`LedgerCloseMetaV1` and `LedgerCloseMetaV2`) defines only a single field: `totalByteSizeOfLiveSorobanState`. There is no separate bucket-list size source in the protocol metadata. The ETL's two-column schema is a backward-compatibility alias for the same underlying value, not an indication that two distinct measurements should diverge. The misleading local variable name `totalByteSizeOfBucketList` is a code-quality issue (likely a remnant from before the XDR field was renamed) but does not produce incorrect data — the code reads the only available source and populates both columns identically, which is the only correct behavior given the data source.

### Lesson Learned

Before hypothesizing that two schema columns should have different values, verify that the upstream XDR protocol actually exposes two distinct source fields. The Stellar protocol evolves and renames fields across versions; ETL schemas may retain old column names as backward-compatible aliases for a single renamed source, not as indicators of two independent measurements.
