# H078: Future transaction-meta version drift is not live in this checkout

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If current production ledgers could contain `TransactionMeta` or `LedgerCloseMeta` versions newer than the transform layer understands, the ETL should still decode those records correctly or fail loudly instead of silently dropping events or fee fields.

## Mechanism

I investigated whether `TransformTransaction()`, `TransformContractEvent()`, and sibling helpers were already stale against a newer generated XDR union arm such as `TransactionMetaV5` or `LedgerCloseMetaV3`. That would have created a concrete wrong-output path if the SDK could deserialize newer meta versions while the transform only branched on older ones.

## Trigger

Feed the current transform layer a ledger whose generated SDK types expose `TransactionMeta.V == 5` or `LedgerCloseMeta.V == 3`, then trace whether the code drops those newer fields.

## Target Code

- `internal/transform/transaction.go:181-211` — Soroban fee/refund extraction only branches on `TransactionMeta` V3 and V4
- `internal/transform/contract_events.go:21-67` — contract-event export relies on SDK `GetTransactionEvents()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_meta.go:7-27` — `GetContractEventsForOperation()` supports only versions 1-4
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_meta.go:35-49` — `GetDiagnosticEvents()` supports only versions 1-4
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/transaction_meta.go:54-62` — `GetTransactionEvents()` supports only versions 1-4
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/ledger_close_meta.go:8-18` — SDK helper supports only `LedgerCloseMeta` V0-V2
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/ledger_close_meta.go:49-59` — transaction counting likewise stops at V2

## Evidence

The transform code clearly has explicit version branches, so stale-union drift is a plausible class of bug in principle. The generated SDK helpers I inspected, however, also stop at `TransactionMeta` V4 and `LedgerCloseMeta` V2; there is no newer union arm in this checkout that the transform is accidentally ignoring.

## Anti-Evidence

Because the generated XDR in the current dependency graph does not define `TransactionMetaV5` or `LedgerCloseMetaV3`, there is no concrete runtime shape that can trigger this suspected omission today. Any bug here would depend on a future SDK/protocol upgrade that is not present in the codebase under analysis.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected newer meta versions do not exist in the current generated XDR types, so the transform cannot encounter them in this checkout. This is future-version hardening work, not a live data-corruption bug.

### Lesson Learned

Before treating version switches as stale, verify the generated SDK unions that actually feed the transform. Missing branches are only actionable when the checked-in XDR already defines a reachable newer arm.
