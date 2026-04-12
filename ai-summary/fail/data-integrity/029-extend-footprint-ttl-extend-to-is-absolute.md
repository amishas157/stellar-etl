# H029: `extend_footprint_ttl` looked like it exported the wrong TTL quantity, but `extend_to` is already the protocol field

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `extend_footprint_ttl`, `history_operations.details.extend_to` should equal the operation's protocol `ExtendTo` value. If the XDR field is already an absolute ledger floor, the ETL should export that number directly rather than deriving some other quantity.

## Mechanism

This looked suspicious because Soroban TTL operations often invite confusion between relative extension amounts and absolute live-until ledgers. But the ETL writes the raw `op.ExtendTo` field directly, and the protocol/docs define that field as the target ledger sequence floor, so the exported scalar matches the source data instead of miscomputing it.

## Trigger

1. Export any `extend_footprint_ttl` operation with a known `ExtendTo` value such as `1234`.
2. Inspect `history_operations.details.extend_to`.
3. Compare it with the operation body and protocol docs; the ETL should emit the same absolute value.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:1144-1152` — assigns `details["extend_to"] = op.ExtendTo`

## Evidence

The live transform path does not derive or offset this value; it copies the XDR field verbatim. The current Stellar docs for `extend_footprint_ttl` describe `extendTo` as the minimum absolute ledger number until which the footprint entries should remain live, which matches the ETL field name and value.

## Anti-Evidence

There is no secondary transformation here that could accidentally reinterpret the field. The only other nearby synthesized values are the footprint-summary fields (`ledger_key_hash`, `contract_id`, `contract_code_hash`), not `extend_to` itself.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

`extend_to` is already the protocol's absolute target-ledger field, and the ETL exports it directly from `op.ExtendTo` without reinterpretation.

### Lesson Learned

For Soroban TTL operations, check the protocol meaning of `ExtendTo` before assuming the exporter should derive a separate "delta" or "live until" value. A raw field copy is only a bug when the protocol meaning differs from the exported column name, which is not the case here.
