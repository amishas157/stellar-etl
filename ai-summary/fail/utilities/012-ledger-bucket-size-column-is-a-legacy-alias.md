# H012: `total_byte_size_of_bucket_list` looks miswired but has no distinct source today

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `ledger.total_byte_size_of_bucket_list` represents a distinct on-chain metric, `TransformLedger()` should populate it from a distinct XDR field rather than mirroring `total_byte_size_of_live_soroban_state`.

## Mechanism

`TransformLedger()` currently assigns `TotalByteSizeOfBucketList` and `TotalByteSizeOfLiveSorobanState` from the same local variable. That initially looked like a copy-paste mapping bug that would make one ledger column claim a bucket-list size even when the source only carried the live Soroban state size.

## Trigger

Export a ledger whose true bucket-list byte size differs from the live Soroban state size.

## Target Code

- `internal/transform/ledger.go:61-91` — reads `TotalByteSizeOfLiveSorobanState` from LCM v1/v2
- `internal/transform/ledger.go:104-128` — writes the same value into both output fields
- `github.com/stellar/go-stellar-sdk/xdr/ledger_close_meta.go:69-77` — SDK helper exposes only `TotalByteSizeOfLiveSorobanState`

## Evidence

The struct literal in `ledger.go` explicitly sets both fields from `outputTotalByteSizeOfLiveSorobanState`. That is the exact shape of a classic copy-paste bug.

## Anti-Evidence

The current XDR definitions exposed by the SDK only provide `TotalByteSizeOfLiveSorobanState`; I did not find any separate `TotalByteSizeOfBucketList` field in the current `LedgerCloseMeta` schema. The helper package in `ingest/ledger/ledger.go` likewise exposes only the live Soroban state metric, which points to a legacy compatibility alias rather than a live wrong-source mapping.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-10
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The code does mirror one value into two differently named columns, but the current protocol/XDR surface does not provide a distinct bucket-list size to preserve. Without a separate on-chain source, this cannot presently produce divergent observable output.

### Lesson Learned

Several Stellar ETL columns retain pre-Protocol-23 names as compatibility aliases. A suspicious duplicate assignment is not sufficient by itself; first verify that the current XDR still exposes two distinct source fields.
