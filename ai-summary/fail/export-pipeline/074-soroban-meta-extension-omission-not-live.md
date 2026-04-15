# H074: Transaction export does not currently miss newer Soroban meta extension arms

**Date**: 2026-04-15
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the current Stellar XDR exposed Soroban transaction metadata in newer extension arms beyond `Ext.V1`, the transaction exporter should surface those live fee/resource values instead of freezing its reads at `GetV1()`.

## Mechanism

`TransformTransaction()` only reads `meta.SorobanMeta.Ext.GetV1()` and the `ResourceExt` arm on `SorobanTransactionDataExt`, which looked like a stale implementation after recent protocol evolution. If the generated unions had already grown additional live arms, newer fee breakdowns or resource metadata would be silently dropped from `history_transactions`.

## Trigger

Export transactions from a ledger whose Soroban transaction metadata uses a hypothetical `SorobanTransactionMetaExt` arm above `V1` or a `SorobanTransactionDataExt` arm beyond `ResourceExt`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:171-175,181-209` — reads only `ResourceExt` and `SorobanMeta.Ext.GetV1()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:17355-17529` — `SorobanTransactionMetaExtV1` and `SorobanTransactionMetaExt`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:34099-34169` — `SorobanTransactionDataExt`

## Evidence

The local transform never probes `GetV2()` or higher on Soroban meta/resource unions, which is often how stale protocol handling first appears after an SDK bump.

## Anti-Evidence

The current generated XDR still defines only `SorobanTransactionMetaExt` arms `0` and `1`, and `SorobanTransactionDataExt` arms `0` and `1` with the single `ResourceExt` payload. There is no live higher arm to miss.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspected omission depends on extension arms that are not present in the current SDK/XDR. The exporter's `GetV1()` reads are stale-looking but still complete for today's protocol surface.

### Lesson Learned

Versioned-union hypotheses need two checks: the transform logic and the current generated XDR. A mapper that only reads `V1` is only a live bug when the union actually exposes a reachable `V2+` arm in the current dependency set.
