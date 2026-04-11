# H043: Ignored `shouldExport` flag leaks nonce contract-data rows

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Medium
**Impact**: Suspected extra-row emission
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `TransformContractData()` returns `shouldExport=false` for nonce entries, the caller should still suppress those rows so nonce-only contract-data keys never reach the export output.

## Mechanism

The command path ignores the third return value from `TransformContractData()`, which initially looked like a classic Go value-loss bug that would let nonce rows flow through despite the helper's explicit skip signal.

## Trigger

Run `export_ledger_entry_changes --export-contract-data` over a range containing nonce-only contract-data entries.

## Target Code

- `internal/transform/contract_data.go:60-63` - nonce rows return `(ContractDataOutput{}, nil, false)`
- `cmd/export_ledger_entry_changes.go:217-230` - caller ignores the returned bool

## Evidence

`TransformContractData()` explicitly treats `ScValTypeScvLedgerKeyNonce` as "should be discarded" and returns `false` (`internal/transform/contract_data.go:60-63`). The caller binds that return to `_` instead of checking it (`cmd/export_ledger_entry_changes.go:217-218`).

## Anti-Evidence

The same caller immediately drops any no-error contract-data row whose `ContractId` is empty (`cmd/export_ledger_entry_changes.go:225-227`). The nonce path returns the zero-value struct, so the empty `ContractId` guard already enforces the skip even though the boolean itself is ignored.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Ignoring the bool is redundant rather than dangerous here: nonce rows come back with an empty `ContractId`, and the caller has a second explicit guard that skips exactly those rows before append.

### Lesson Learned

An ignored return value is only actionable if no other guard preserves the contract. Always trace the complete call site before turning a discarded bool into a live data-loss claim.
