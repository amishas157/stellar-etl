# H001: Native SAC balance rows bypass the lumens exclusion guard

**Date**: 2026-04-11
**Subsystem**: data-integrity (transform)
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes --export-contract-data` encounters a `ContractData` balance row belonging to the native-asset contract, it should leave `contract_data_balance_holder` and `contract_data_balance` empty instead of treating the row as a supported SAC token balance. The upstream SAC helper explicitly excludes lumens asset stats, so native balance rows should not surface as normal `soroban_contract_data` balance facts.

## Mechanism

`ContractBalanceFromContractData()` computes the native asset contract ID but never compares it against `contractData.Contract.ContractId`, so native balance rows fall through the normal SAC-balance parser. `TransformContractData()` then serializes those rows with populated `contract_data_balance_holder` and `contract_data_balance`, even though the reference helper this code was copied from rejects lumens rows before any balance extraction.

## Trigger

Export contract-data changes across a ledger containing a persistent native-asset contract balance entry keyed like `["Balance", ScAddress(contract)]` under `xdr.MustNewNativeAsset().ContractID(passphrase)`.

## Target Code

- `internal/transform/contract_data.go:84-100` — `TransformContractData()` copies any non-nil parsed balance into `contract_data_balance_holder` and `contract_data_balance`.
- `internal/transform/contract_data.go:306-378` — `ContractBalanceFromContractData()` parses balance rows, computes `nativeAssetContractID`, and never rejects it.

## Evidence

The local helper calls `xdr.MustNewNativeAsset().ContractID(passphrase)` at `contract_data.go:312` but discards the returned ID instead of checking whether the current row belongs to that contract. The upstream SAC implementation at `go-stellar-sdk/ingest/sac/contract_data.go:211-214` performs exactly that guard and returns `false` for lumens rows.

## Anti-Evidence

If stellar-etl intentionally wants broader native-asset coverage than the upstream SAC helper, this could be an intentional divergence. But there is no matching test or code comment in the local copy documenting that policy, and the unused native contract ID strongly suggests the rejection logic was dropped accidentally.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced `ContractBalanceFromContractData` (contract_data.go:306-378) and compared it line-by-line against the upstream `go-stellar-sdk@v0.1.0/ingest/sac/contract_data.go`. The upstream version assigns the native asset contract ID and performs an explicit guard: `if err != nil || (contractData.Contract.ContractId != nil && *contractData.Contract.ContractId == nativeAssetContractID) { return ..., false }`. The local copy discards the ID to `_` (line 312) and only checks for errors, allowing native balance rows to pass through. `TransformContractData` at line 96-101 then populates `contract_data_balance_holder` and `contract_data_balance` for any row where `dataBalance != nil`, so native balance rows emit non-empty balance fields.

### Code Paths Examined

- `internal/transform/contract_data.go:306-315` — `ContractBalanceFromContractData` computes native contract ID via `xdr.MustNewNativeAsset().ContractID(passphrase)` but assigns to `_`, discarding the result. Only `err` is checked. No comparison against `contractData.Contract.ContractId`.
- `go-stellar-sdk@v0.1.0/ingest/sac/contract_data.go:211-214` — Upstream version keeps the ID: `nativeAssetContractID, err := ...` and guards with `if err != nil || (...ContractId == nativeAssetContractID)`. Comment reads "we don't support asset stats for lumens."
- `internal/transform/contract_data.go:96-101` — `TransformContractData` calls `t.ContractBalanceFromContractData(ledgerEntry, passphrase)` and populates output fields unconditionally when `dataBalance != nil`.
- `internal/transform/contract_data.go:191-297` — Local `AssetFromContractData` **intentionally** diverges from upstream to handle `"Native"` case (line 243-247), but this is for contract-instance asset-info entries, not balance entries.
- `internal/transform/contract_data_test.go:61-76` — Tests use mocked `MockContractBalanceFromContractData` and never exercise the real native-asset guard logic.
- `go-stellar-sdk@v0.1.0/ingest/sac/contract_data.go:208-210` — Upstream also checks `contractData.Durability != xdr.ContractDataDurabilityPersistent`, another guard missing from the local version (minor, as SAC balances are always persistent).

### Findings

The bug is confirmed. The local `ContractBalanceFromContractData` was derived from the upstream implementation but the native asset exclusion guard was accidentally dropped. Key evidence:

1. **Dead computation**: Line 312 computes `xdr.MustNewNativeAsset().ContractID(passphrase)` but assigns to `_`. This has no purpose other than passphrase validation, which is already handled by the error check. The computation only makes sense as the first half of a comparison that was removed.

2. **Upstream explicitly guards**: The upstream function has an inline comment "we don't support asset stats for lumens" and rejects native balance rows in the same expression that checks for errors.

3. **Intentional vs accidental divergence**: The local `AssetFromContractData` has a clearly intentional `"Native"` case with custom logic (lines 243-247). In contrast, `ContractBalanceFromContractData` has no native-specific handling at all — just a discarded computation. If the divergence were intentional, there would be a comment, a test, or custom logic analogous to the `AssetFromContractData` native case.

4. **No test coverage**: The test file uses mocked functions that never exercise the real native-asset code path, so this gap was never caught.

**Impact**: Every native-asset SAC balance entry in the ledger emits a `soroban_contract_data` row with non-empty `contract_data_balance_holder` and `contract_data_balance` fields. Downstream consumers that aggregate SAC token balances (e.g., for asset supply metrics) would silently include native XLM balances they are not designed to handle.

### PoC Guidance

- **Test file**: `internal/transform/contract_data_test.go`
- **Setup**: Use `sac.BalanceToContractData(nativeContractID, holderID, amount)` from `go-stellar-sdk/ingest/sac` to create a native-asset balance ledger entry where `nativeContractID` is computed from `xdr.MustNewNativeAsset().ContractID(passphrase)`. Create a `TransformContractDataStruct` initialized with the **real** `ContractBalanceFromContractData` (not the mock).
- **Steps**: Call `ContractBalanceFromContractData(nativeBalanceEntry, passphrase)` directly. Then call `TransformContractData` with a change containing the native balance entry.
- **Assertion**: Assert that `ContractBalanceFromContractData` returns `false` for native balance entries (currently it returns `true`). Assert that the transformed output has empty `ContractDataBalanceHolder` and `ContractDataBalance` fields for native entries.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestNativeSACBalanceBypassesLumensGuard"
**Test Language**: Go

### Demonstration

The test constructs a native-asset SAC balance ledger entry using `sac.BalanceToContractData` with the native contract ID derived from `xdr.MustNewNativeAsset().ContractID(passphrase)`. It then calls the real `ContractBalanceFromContractData` and observes that it returns `ok=true` with `balance=1000000`, proving that native balance rows are accepted rather than rejected. The upstream SDK version rejects these rows with an explicit guard that was accidentally dropped in the local copy.

### Test Body

```go
func TestNativeSACBalanceBypassesLumensGuard(t *testing.T) {
	const passphrase = "Test SDF Network ; September 2015"

	// 1. Compute the native-asset contract ID (same as line 312 of contract_data.go)
	nativeContractID, err := xdr.MustNewNativeAsset().ContractID(passphrase)
	if err != nil {
		t.Fatalf("failed to compute native asset contract ID: %v", err)
	}

	// 2. Build a balance entry keyed to the native-asset contract
	holderID := [32]byte{0xAA}
	amount := uint64(1_000_000)
	balanceData := sac.BalanceToContractData(nativeContractID, holderID, amount)

	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data:                  balanceData,
	}

	// 3. Call the real (non-mocked) ContractBalanceFromContractData
	_, balance, ok := ContractBalanceFromContractData(ledgerEntry, passphrase)

	// 4. The upstream SAC helper returns false for native balance rows.
	//    The local copy returns true — demonstrating the bug.
	if ok && balance != nil {
		t.Errorf("BUG CONFIRMED: ContractBalanceFromContractData accepted a native-asset "+
			"balance row (ok=%v, balance=%s). "+
			"Expected ok=false for native/lumens balance entries, matching upstream behavior.",
			ok, balance.String())
	} else {
		t.Logf("Native balance row correctly rejected (ok=%v)", ok)
	}
}
```

### Test Output

```
=== RUN   TestNativeSACBalanceBypassesLumensGuard
    data_integrity_poc_test.go:102: BUG CONFIRMED: ContractBalanceFromContractData accepted a native-asset balance row (ok=true, balance=1000000). Expected ok=false for native/lumens balance entries, matching upstream behavior.
--- FAIL: TestNativeSACBalanceBypassesLumensGuard (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.797s
```
