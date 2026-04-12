# H002: `contract_data` treats arbitrary balance-shaped storage as SAC balances

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`contract_data_balance_holder` and `contract_data_balance` should only be
populated for genuine Stellar Asset Contract balance rows. A custom contract
whose storage merely resembles the SAC `Balance` layout should export those
columns as empty, because the row does not represent a canonical asset-contract
balance.

## Mechanism

`TransformContractData()` blindly trusts `ContractBalanceFromContractData()`
whenever it returns a holder and amount. Unlike `AssetFromContractData()`,
which recomputes the expected asset contract ID and rejects forged asset-info
rows, `ContractBalanceFromContractData()` only validates the key/value shape
(`["Balance", <address>]` plus `{amount, authorized, clawback}`) and never
verifies that the owning `contractData.Contract.ContractId` is actually a
Stellar Asset Contract. Any non-SAC contract that stores a balance map in that
shape will therefore be exported as if it were SAC balance state.

## Trigger

Export `contract_data` for a legitimate custom contract that stores persistent
entries with key `["Balance", <contract-address>]` and a value map containing
`amount`, `authorized`, and `clawback`. The resulting row will populate
`contract_data_balance_holder` / `contract_data_balance` even though the
contract is not a Stellar Asset Contract.

## Target Code

- `internal/transform/contract_data.go:84-101` — transform populates exported balance columns whenever the helper returns a value
- `internal/transform/contract_data.go:191-296` — sibling asset-info parser performs an explicit contract-ID integrity check
- `internal/transform/contract_data.go:306-378` — balance parser validates only storage shape and amount sign, not SAC identity
- `internal/transform/contract_data_test.go:61-76` — tests inject a mock balance parser instead of exercising the real parser logic

## Evidence

The contrast inside the same file is strong: asset-info parsing includes an
anti-forgery contract-ID check, while balance parsing does not. The balance
helper even computes the native asset contract ID at line 312, but never uses
that or any other identity proof before returning `true`.

## Anti-Evidence

This only manifests when a non-SAC contract stores data in a SAC-like shape;
genuine SAC rows still parse correctly. But the threat model here is legitimate
chain data from arbitrary contracts, and the exported balance columns would look
plausible while pointing at the wrong contract class entirely.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `ContractBalanceFromContractData` in the ETL (`internal/transform/contract_data.go:306-378`) and confirmed it performs shape-only matching without SAC identity verification. Then traced the upstream canonical implementation in `go-stellar-sdk/ingest/sac/contract_data.go:203-275` and found the identical pattern: shape-only matching with no SAC identity check. The asymmetry between `AssetFromContractData` (has identity check) and `ContractBalanceFromContractData` (no identity check) exists in both the ETL and the upstream SDK, establishing this as the ecosystem convention.

### Code Paths Examined

- `internal/transform/contract_data.go:96-101` — `TransformContractData` populates balance fields whenever `ContractBalanceFromContractData` returns non-nil; no additional SAC verification
- `internal/transform/contract_data.go:306-378` — ETL's balance parser validates key shape (`["Balance", <address>]`), value shape (`{amount, authorized, clawback}`), and amount sign, but no SAC identity check; line 312 computes native asset contract ID and discards it (`_`)
- `internal/transform/contract_data.go:191-296` — ETL's `AssetFromContractData` does have the anti-forgery identity check at lines 288-293
- `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:203-275` — Upstream SDK's canonical `ContractBalanceFromContractData` has the same shape-only matching; adds durability check and native asset exclusion but no SAC identity verification
- `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:170-176` — Upstream SDK's `AssetFromContractData` also has the identity check, confirming the asymmetry exists at the SDK level
- `cmd/export_ledger_entry_changes.go:217` — Production wiring passes the real `transform.ContractBalanceFromContractData` function

### Why It Failed

The upstream SDK's canonical `ContractBalanceFromContractData` (`go-stellar-sdk/ingest/sac/contract_data.go:203-275`) uses the same shape-only matching convention without SAC identity verification. The asymmetry between `AssetFromContractData` (has identity check) and `ContractBalanceFromContractData` (no identity check) is shared between the ETL and the upstream SDK. Per established meta-pattern: when a suspicious transform behavior mirrors the canonical upstream SDK processor, it is an SDK-contract convention, not an ETL-specific bug. The ETL does not add extra corruption on top of the upstream pattern; it faithfully reproduces the same behavior.

### Lesson Learned

When an ETL function's own implementation lacks a guard that a sibling function has, check whether the canonical upstream SDK implementation also lacks the same guard. If the upstream SDK's equivalent function shares the identical gap, the behavior is the established ecosystem convention. The `AssetFromContractData` / `ContractBalanceFromContractData` asymmetry is a design choice at the SDK level, not an ETL-local omission.
