# H002: Temporary contract balances are exported as persistent SAC balances

**Date**: 2026-04-11
**Subsystem**: data-integrity (transform)
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Only persistent SAC balance entries should populate `contract_data_balance_holder` and `contract_data_balance`. Temporary `ContractData` rows should remain generic contract-data exports with those balance columns blank, because temporary storage is not durable token balance state.

## Mechanism

The local `ContractBalanceFromContractData()` omits the upstream `contractData.Durability != Persistent` guard, so any temporary row whose key/value shape happens to match the SAC balance layout is accepted as a real token balance. `TransformContractData()` then emits a plausible balance number and holder contract ID for ephemeral storage that should never be interpreted as a durable asset position.

## Trigger

Export contract-data changes for a contract that writes a temporary `ContractData` row with key `["Balance", ScAddress(contract)]` and value map `{amount, authorized, clawback}`. The row will be exported with populated balance columns even though its durability is temporary.

## Target Code

- `internal/transform/contract_data.go:96-100` — `TransformContractData()` turns any parsed balance into exported `contract_data_balance*` fields without re-checking durability.
- `internal/transform/contract_data.go:306-378` — `ContractBalanceFromContractData()` never checks `contractData.Durability`.

## Evidence

The upstream helper at `go-stellar-sdk/ingest/sac/contract_data.go:208-210` immediately rejects non-persistent rows before parsing the balance key. The local copy removed that guard entirely, even though the function comment still says it verifies entries written by the Stellar Asset Contract.

## Anti-Evidence

Canonical SAC balance storage is persistent today, so this requires a contract that uses the same balance-shaped layout in temporary storage. But `TransformContractData()` is a general contract-data exporter, not a SAC-only endpoint, so accepting temporary rows as durable balances is still a live misclassification path.
