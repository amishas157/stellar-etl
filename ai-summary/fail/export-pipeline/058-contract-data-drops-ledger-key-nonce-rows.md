# H003: Contract-data export silently drops `LedgerKeyNonce` rows

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --export-contract-data` should emit one
`soroban_contract_data` row for every `LedgerEntryTypeContractData` change in
the selected ledger range, including entries whose key type is
`ScValTypeScvLedgerKeyNonce`. For nonce-backed rows, the correct output should
preserve the contract ID, key type, serialized key/value, XDR blob, and ledger
key hashes rather than dropping the row entirely.

## Mechanism

`TransformContractData()` hard-codes a guard that returns `(ContractDataOutput{},
nil, false)` whenever `contractData.Key.Type` is `ScValTypeScvLedgerKeyNonce`.
`export_ledger_entry_changes` then discards that row before append, so the
contract-data export becomes a filtered subset of `LedgerEntryTypeContractData`
changes without any CLI flag, schema field, or documentation indicating that
nonce rows are excluded.

## Trigger

1. Find a ledger batch containing a `LedgerEntryTypeContractData` change whose
   key type is `ScValTypeScvLedgerKeyNonce`.
2. Run `export_ledger_entry_changes --export-contract-data`.
3. Compare the exported rows against the source ledger changes.
4. The nonce-backed contract-data entry will be absent from the output even
   though it is a real `ContractData` ledger entry in the input stream.

## Target Code

- `internal/transform/contract_data.go:49-63` — returns early for
  `ScValTypeScvLedgerKeyNonce`
- `internal/transform/schema.go:516-537` — schema describes a generic
  `ContractDataOutput` row with no documented nonce exclusion
- `cmd/export_ledger_entry_changes.go:218-230` — caller receives the transform
  result and only appends rows that survive that early return
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go`
  — `ScValTypeScvLedgerKeyNonce` is a real `ScVal` arm, not an impossible type

## Evidence

The drop is explicit and unconditional: the transform reaches the real
`ContractData` row, checks the key type, and discards it before any schema
mapping happens. The export flag and schema are both named generically
(`contract-data`, `soroban_contract_data`) rather than "non-nonce contract
data", so the current behavior is a silent narrowing of the exported dataset.

## Anti-Evidence

The code comment says nonce rows "should be discarded", which suggests this may
reflect an intended product decision rather than an accidental omission. Review
should confirm whether nonce-backed contract-data entries are considered
user-visible contract state or merely internal ledger machinery that this table
is intentionally allowed to skip.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete path from `export_ledger_entry_changes` through `TransformContractData()`. The nonce filtering occurs at two layers: the transform function at `contract_data.go:60-62` returns `(ContractDataOutput{}, nil, false)` for `ScValTypeScvLedgerKeyNonce` entries, and the caller at `export_ledger_entry_changes.go:225-228` independently checks for the empty result with the comment "Empty contract data that has no error is a nonce. Does not need to be recorded." Both layers explicitly acknowledge and intentionally implement this filtering.

### Code Paths Examined

- `internal/transform/contract_data.go:60-62` — explicit `ScValTypeScvLedgerKeyNonce` check with comment "Is a nonce and should be discarded", returns zero struct with nil error and false
- `cmd/export_ledger_entry_changes.go:216-231` — caller discards the third return value but checks `contractData.ContractId == ""` with comment "Empty contract data that has no error is a nonce. Does not need to be recorded"

### Why It Failed

This is **working-as-designed behavior**, not a bug. The filtering is intentional at two independent code layers with explicit comments explaining the rationale. `ScValTypeScvLedgerKeyNonce` entries are internal Soroban protocol machinery used for authorization signature replay protection — they are not user-meaningful contract state. They contain only a nonce counter value, have no semantic relationship to the contract's business logic, and are transient bookkeeping entries created by the runtime. Excluding them from the `soroban_contract_data` export is a deliberate product decision to keep the export focused on meaningful contract storage, consistent with how other Stellar data pipeline tools handle nonce entries.

### Lesson Learned

When a hypothesis identifies an explicit code guard with a clear comment explaining the intent, the anti-evidence section deserves priority investigation. Two-layer filtering (transform + caller) with matching comments is a strong signal of intentional design rather than accidental omission. Nonce entries in Soroban are internal protocol state, not user contract data.
