# H032: `export_ledgers` duplicates live Soroban state into `total_byte_size_of_bucket_list`

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong ledger-size field mapping
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the export schema exposes both `total_byte_size_of_bucket_list` and `total_byte_size_of_live_soroban_state`, the two columns should come from distinct protocol values whenever the underlying XDR exposes distinct source fields.

## Mechanism

`TransformLedger()` assigns both output columns from `TotalByteSizeOfLiveSorobanState`, which initially looks like a copy-paste mapping bug. If current close-meta XDR still carried an independent bucket-list-size field, this would silently alias two different metrics into one exported value.

## Trigger

Export a ledger whose true bucket-list size and live-Soroban-state size differ, then compare the two exported columns.

## Target Code

- `internal/transform/ledger.go:71-86` — reads only `TotalByteSizeOfLiveSorobanState`
- `internal/transform/ledger.go:125-126` — writes that one value into both exported columns
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:19600-19607` — `LedgerCloseMetaV1` fields
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:19855-19862` — `LedgerCloseMetaV2` fields

## Evidence

The transform code really does duplicate one source field into two output columns, and there is no second read of any `TotalByteSizeOfBucketList` source value anywhere in the transform path.

## Anti-Evidence

Current XDR does **not** expose a distinct bucket-list-size field in `LedgerCloseMetaV1` or `LedgerCloseMetaV2`; both structs only carry `TotalByteSizeOfLiveSorobanState`. Without a separate on-chain source, this is a legacy compatibility alias, not a data-conversion bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected copy-paste bug depends on a distinct bucket-list-size source value, but the current SDK/XDR types do not provide one. The exporter cannot preserve a value that the upstream protocol no longer exposes separately.

### Lesson Learned

Legacy-looking duplicate columns are only real corruption if the source protocol still contains distinct fields. For ledger-size audits, the decisive check is the current `LedgerCloseMeta` definition, not the schema column names alone.
