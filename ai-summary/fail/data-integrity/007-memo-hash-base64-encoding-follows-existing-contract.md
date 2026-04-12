# H007: Memo Hash Base64 Encoding Follows Existing Contract

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Low
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the ETL promised a specific textual encoding for `memo` when the memo type
is `hash` or `return`, then the exporter should serialize those 32-byte values
using that documented encoding consistently.

## Mechanism

`TransformTransaction()` base64-encodes `MemoHash` and `MemoReturn` bytes
before storing them in `memo`. I investigated whether this silently corrupts
the output by choosing the wrong textual representation for those raw bytes.

## Trigger

1. Export any transaction with `MemoTypeMemoHash` or `MemoTypeMemoReturn`.
2. Inspect `history_transactions.memo`.
3. Observe that the ETL emits base64 text for the memo bytes.

## Target Code

- `internal/transform/transaction.go:81-86` — base64-encodes memo hash bytes
- `.../go-stellar-sdk/ingest/ledger_transaction.go:364-369` — upstream helper
  uses the same base64 encoding

## Evidence

The memo bytes are not hex-encoded or XDR-encoded; the ETL explicitly chooses
base64 text.

## Anti-Evidence

The ETL exactly matches the upstream ingest helper, and I found no competing
repository contract that says memo hash bytes should be exported differently.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

I could not establish that base64 is wrong for this ETL contract; I only
established that it is the chosen representation. Since the local code and the
upstream helper agree, and there is no repo evidence requiring another
encoding, this is not a demonstrated correctness bug.

### Lesson Learned

Encoding choices are only integrity bugs when there is a concrete conflicting
contract. Matching the ETL's upstream helper is strong anti-evidence unless the
repository explicitly documents a different representation.
