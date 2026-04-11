# H001: Fee-bump `tx_signers` drops the inner transaction authorizers

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a fee-bump transaction, the exported `tx_signers` field should preserve the signer set that authorized the inner transaction payload, and it should not replace those signatures with the outer fee-payer signatures. When the inner transaction and the outer fee-bump wrapper are signed by different keys, downstream consumers should be able to recover the operation-authorizing signer set from the transaction row.

## Mechanism

`TransformTransaction()` first populates `TxSigners` from `transaction.Envelope.Signatures()`. In the SDK, that helper returns the **inner** signatures for fee-bump envelopes. The fee-bump branch then calls `getTxSigners(transaction.Envelope.FeeBump.Signatures)` and overwrites `output.TxSigners`, so the final row silently drops the inner authorizers and retains only the outer fee-account signatures.

## Trigger

Export any ledger range containing a fee-bump transaction where the inner transaction is signed by account A and the outer fee-bump wrapper is signed by a different fee-payer account B. The JSON row will contain only B's signatures in `tx_signers`, even though the wrapped operations were authorized by A.

## Target Code

- `internal/transform/transaction.go:227-300` — initializes `TxSigners`, then overwrites it inside the fee-bump branch
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/transaction_envelope.go:67-80` — `TransactionEnvelope.Signatures()` returns inner signatures for fee-bump envelopes
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/transaction_envelope.go:189-197` — fee-bump signatures are exposed separately from the inner signature set

## Evidence

The live code already treats `transaction.Envelope.Signatures()` as the initial source of `tx_signers`, which matches the SDK helper's fee-bump behavior. The later overwrite in the fee-bump block discards that first result instead of merging or preserving it, so a transaction with distinct inner and outer signatures will export only the fee-payer layer.

## Anti-Evidence

If the inner and outer envelopes are signed by the same key, or if test fixtures only exercise one signature layer, the bug is masked because the final `tx_signers` slice still looks plausible. The field name also does not spell out whether it intends to represent inner-only, outer-only, or full-envelope signers, so the intended contract needs reviewer confirmation.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (distinct from success/001 which covers the encoding bug, not the source-selection question)
**Failed At**: reviewer

### Trace Summary

Traced the full fee-bump path in `TransformTransaction()`. Line 227 calls `getTxSigners(transaction.Envelope.Signatures())`, which for fee-bump envelopes returns the inner V1 signatures via the SDK helper. Lines 295-300 then overwrite `TxSigners` with `getTxSigners(transaction.Envelope.FeeBump.Signatures)` — the outer fee-bump signatures. This overwrite is **intentional** and matches Horizon's API convention where the top-level `signatures` field on a fee-bump transaction contains the outer fee-bump signatures, not the inner authorizers.

### Code Paths Examined

- `internal/transform/transaction.go:227-230` — initial `TxSigners` from `Envelope.Signatures()` (inner for fee-bump)
- `internal/transform/transaction.go:284-301` — fee-bump branch overwrites `TxSigners` with outer `FeeBump.Signatures`
- `go-stellar-sdk/.../xdr/transaction_envelope.go:78-88` — `Signatures()` returns inner signatures for fee-bump; `FeeBumpSignatures()` returns outer
- `internal/transform/transaction_test.go:143,216` — test cases explicitly assert `TxSigners` = outer fee-bump signatures
- `internal/transform/transaction_test.go:308-356` — fee-bump test fixture has `FeeBump.Signatures` set, inner `V1.Signatures` is empty

### Why It Failed

The hypothesis describes **working-as-designed behavior**, not a bug. The "expected behavior" stated in the hypothesis is incorrect:

1. **Horizon convention**: In Horizon's API, a fee-bump transaction's top-level `signatures` field contains the **outer** fee-bump signatures. The inner transaction signatures are accessible via the inner_transaction sub-object. The ETL follows this same convention.

2. **Consistent field semantics**: The ETL's fee-bump field split is internally consistent with Horizon — `Account` = inner source, `FeeAccount` = outer source, `TxSigners` = outer signatures. This matches `source_account`/`fee_account`/`signatures` in Horizon.

3. **Tests assert the behavior**: `transaction_test.go` explicitly expects `TxSigners` to contain the outer fee-bump signatures (lines 143, 216). This is authoritative evidence of design intent.

4. **Related fail entries confirm the pattern**: Fail entry 006 established that SDK helpers correctly handle fee-bump disambiguation. Fail entry 033 confirmed the intentional inner/outer split for fee fields.

### Lesson Learned

For fee-bump transactions, the ETL intentionally follows Horizon's convention: `tx_signers` contains the outer fee-bump envelope's signatures, not the inner transaction's authorizers. Before claiming a field uses the "wrong" set of fee-bump data, verify which convention Horizon uses for the equivalent API field.
