# H027: `restored_key.ledger_entry_type` looked like raw SDK enum leakage, but tests make it contractual

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `restored_key.ledger_entry_type` is meant to align with the ETL's other entity-type columns, it should export a normalized value such as `contract_data` or `ttl`, not raw SDK enum names like `LedgerEntryTypeContractData`.

## Mechanism

`TransformRestoredKey()` derives the field with `key.Type.String()`, which initially looked like the same class of bug as other raw-XDR-enum leaks in this repository. That would have made restored-key rows structurally inconsistent with the ETL's usual lowercase canonical naming.

## Trigger

1. Export restored-key rows from a ledger containing `ContractData` or `Ttl` restored entries.
2. Inspect `ledger_entry_type`.
3. Observe values like `LedgerEntryTypeContractData` and `LedgerEntryTypeTtl`.

## Target Code

- `internal/transform/restored_key.go:TransformRestoredKey:12-46` — sets `ledgerEntryType := key.Type.String()`
- `internal/transform/restored_key_test.go:84-91` — unit test fixture expects `LedgerEntryTypeOffer`
- `testdata/changes/restored_key.golden:1-2` — golden rows expect `LedgerEntryTypeContractData` and `LedgerEntryTypeTtl`

## Evidence

The production transformer clearly uses the raw XDR enum string, and that is the kind of naming leak that has produced real integrity bugs elsewhere in the ETL.

## Anti-Evidence

Both the dedicated unit test and the checked-in golden output assert the raw enum names explicitly, which strongly indicates this is the repository's intended contract for restored-key exports rather than an accidental mismatch.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The suspicious raw-enum representation is locked in by both unit tests and golden fixtures. Whatever the broader naming conventions elsewhere in the ETL, `restored_key.ledger_entry_type` is intentionally exported as the SDK enum string today.

### Lesson Learned

For output-contract questions, checked-in tests and goldens outweigh stylistic consistency arguments. If fixtures already assert the suspicious value literally, treat it as contractual unless another live surface contradicts it.
