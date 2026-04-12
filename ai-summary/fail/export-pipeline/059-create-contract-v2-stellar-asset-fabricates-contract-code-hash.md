# H002: `create_contract_v2` with `StellarAsset` executable can fabricate an unrelated `contract_code_hash`

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `history_operations` rows where `type=invoke_host_function` and
`details.type="create_contract_v2"`, if `CreateContractArgsV2.Executable.Type`
is `ContractExecutableTypeContractExecutableStellarAsset`, the exported
`details.contract_code_hash` should stay empty/null. That executable arm has no
per-operation Wasm hash, so unrelated footprint `ContractCode` keys should not
be promoted into the row.

## Mechanism

Both create-contract-v2 detail builders ignore `args.Executable.Type` and
always fill `details["contract_code_hash"]` with
`contractCodeHashFromTxEnvelope(transactionEnvelope)`. When a legitimate
Stellar-asset deployment transaction also carries some other `ContractCode`
ledger key in its footprint (for example from a contract-driven deployment
context), the exporter can emit a syntactically valid 64-byte hash that does
not belong to the created asset contract at all.

## Trigger

1. Construct or locate a `create_contract_v2` Soroban operation whose
   `Executable.Type` is `ContractExecutableTypeContractExecutableStellarAsset`.
2. Ensure the transaction footprint also includes an unrelated
   `LedgerKeyContractCode` entry ahead of any other code keys.
3. Run `export_operations`.
4. The exported row will populate `details.contract_code_hash` from the first
   footprint code key even though the operation's executable arm has no Wasm
   hash of its own.

## Target Code

- `internal/transform/operation.go:1119-1139` — free-function
  `create_contract_v2` branch always populates `contract_code_hash` from the
  footprint helper
- `internal/transform/operation.go:1741-1761` — duplicated method-receiver path
  repeats the same logic
- `internal/transform/operation.go:1841-1857` — footprint helper returns the
  first `ContractCode` key it finds
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:28913-28916`
  — `CreateContractArgsV2` carries `Executable ContractExecutable`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:56389-56425`
  — the `StellarAsset` executable arm is void and has no Wasm hash field
- `internal/transform/operation_test.go:2001-2032` — current tests already
  expect an empty `contract_code_hash` for an asset-based `create_contract_v2`
  case when no footprint code key is present

## Evidence

The XDR union proves the key invariant: the `StellarAsset` executable arm has no
`WasmHash`, so there is no operation-scoped code hash to export. The current
operation tests already encode the "empty hash" behavior for asset-based
`create_contract_v2` rows, which means any non-empty hash sourced from an
unrelated footprint key would be fabricated data rather than a lossy summary of
the operation itself.

## Anti-Evidence

This bug stays masked when the transaction footprint contains no
`LedgerKeyContractCode` entries at all, in which case the helper returns an
empty string and the row happens to look correct. Review should verify that
real-world Stellar-asset deployments can legitimately carry unrelated code keys
in the same footprint, rather than only in synthetic test setups.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — near-duplicate of fail/export-pipeline/052-upload-wasm-contract-code-hash-follows-footprint-order.md
**Failed At**: reviewer

### Trace Summary

Both `create_contract_v2` code paths (lines 1119-1127 free-function, 1741-1749 method-receiver) unconditionally call `contractCodeHashFromTxEnvelope(transactionEnvelope)` without checking `args.Executable.Type`. The helper (lines 1841-1857) scans ReadOnly then ReadWrite footprint for any `LedgerKeyContractCode` and returns the first match. For `StellarAsset` executables, this could theoretically emit a non-empty hash where none belongs. However, this is the same `contractCodeHashFromTxEnvelope` helper and the same precondition (unrelated `ContractCode` key in footprint) that was investigated and rejected in H052.

### Code Paths Examined

- `internal/transform/operation.go:1119-1127` — `create_contract_v2` free-function: unconditionally calls `contractCodeHashFromTxEnvelope`, confirmed no `Executable.Type` check
- `internal/transform/operation.go:1741-1749` — method-receiver variant: identical pattern
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope`: scans ReadOnly then ReadWrite, returns first `ContractCode` hash
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData`: extracts hex hash from `LedgerKeyContractCode`
- `internal/transform/operation_test.go:589-621` — existing test uses `Executable: xdr.ContractExecutable{}` (zero-value = Wasm type, not StellarAsset) with no footprint, so `contract_code_hash` is naturally empty

### Why It Failed

This hypothesis is a near-duplicate of H052 (`upload_wasm` contract_code_hash follows footprint order), which exercised the same `contractCodeHashFromTxEnvelope` helper and was rejected at final review because the precondition — an unrelated `ContractCode` key in the transaction footprint — was not demonstrated to be reachable on-chain. The same precondition applies here: Soroban transactions have footprints prepared from simulation containing only actually-accessed keys, and this hypothesis provides no evidence that a `StellarAsset` deployment can legitimately carry an unrelated `ContractCode` entry. Additionally, the existing test at line 610 uses `Executable: xdr.ContractExecutable{}` which is the Wasm zero-value (type 0), not `StellarAsset` (type 1), so the claim that "tests already expect empty hash for asset-based create_contract_v2" is imprecise — the test uses a default-initialized Wasm executable, not an explicit StellarAsset one.

### Lesson Learned

Hypotheses targeting the same shared helper function (`contractCodeHashFromTxEnvelope`) with the same unresolved precondition (unrelated ContractCode key in footprint) are substantially equivalent regardless of which operation type triggers them. The realism of the footprint precondition must be established before exploring variations across operation types.
