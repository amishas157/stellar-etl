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
