# H051: Token-transfer numeric `to_muxed` reconstruction double-wraps an already-muxed destination

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: destination identity corruption in token-transfer exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When upstream token-transfer metadata carries a numeric `ToMuxedInfo` ID for a
destination account, the ETL should reconstruct the original `M...` destination
exactly once. It should not take an already-muxed `to` string, treat it as a base
account ID, and synthesize a second incorrect muxed address.

## Mechanism

At first glance, `transformEvents()` looks risky because it takes `to.String`,
feeds it through `strkey.MuxedAccount.SetAccountID(...)`, then applies the numeric
ID from `ToMuxedInfo`. If the upstream `to` field were already an `M...` address,
this would indeed double-wrap the destination and export a wrong `to_muxed`.

## Trigger

1. Create a token-transfer or mint event with numeric `ToMuxedInfo`.
2. Assume the event's `to` field is already encoded as the final `M...` address.
3. Run `transformEvents()` and inspect the reconstructed `to_muxed`.

## Target Code

- `internal/transform/token_transfer.go:98-105` — reconstructs `to_muxed` from `to.String` plus `ToMuxedInfo.GetId()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/helpers.go:61-63` — upstream helper canonicalizes `MuxedAccount` to the base `G...` account string
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/token_transfer_processor_test.go:239-250` — test helpers attach `ToMuxedInfo` separately from the canonical `to` address string

## Evidence

The ETL reconstruction path is real and would be wrong if `to` already carried an
`M...` address. The upstream processor also stores numeric muxed metadata out of
band in `ToMuxedInfo`, which makes this shape superficially plausible.

## Anti-Evidence

The upstream token-transfer processor does **not** populate `to` with an `M...`
address in this path. Its `protoAddressFromAccount()` helper explicitly converts a
`MuxedAccount` to `account.ToAccountId().Address()`, i.e. the canonical base
`G...` account, and then supplies the muxed numeric ID separately via
`ToMuxedInfo`. That means stellar-etl's reconstruction logic is using the correct
inputs for numeric-ID cases rather than double-wrapping an already-muxed address.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The feared double-mux path depends on upstream `TokenTransferEvent.To` already
containing an `M...` address. The current upstream processor does the opposite:
it canonicalizes the destination to the base `G...` account and carries the muxed
numeric ID separately in `ToMuxedInfo`, so ETL's reconstruction is correct for
numeric-ID cases.

### Lesson Learned

For muxed-address hypotheses, inspect the upstream processor's event-construction
helpers before assuming the string field and the sidecar muxed metadata encode the
same representation. In this pipeline, the string and the muxed-info fields are
intentionally complementary rather than redundant.
