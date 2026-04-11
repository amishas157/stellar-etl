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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (distinct from reviewed/001 which covers the missing native-asset exclusion guard in the same function)

### Trace Summary

I compared the local `ContractBalanceFromContractData` (contract_data.go:306-379) line-by-line against the upstream SDK version at `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:203-275`. The upstream version has an explicit durability guard at lines 208-210 that rejects any entry where `contractData.Durability != xdr.ContractDataDurabilityPersistent`. The local copy completely omits this check. `TransformContractData` at line 96-101 then unconditionally populates `contract_data_balance_holder` and `contract_data_balance` whenever `dataBalance != nil`, so temporary entries with balance-shaped data will have these fields populated in exports.

### Code Paths Examined

- `internal/transform/contract_data.go:306-315` — `ContractBalanceFromContractData` entry: extracts `contractData` from ledger entry, computes native asset contract ID (discarded to `_`), but **no durability check** exists between data extraction and key parsing.
- `go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/sac/contract_data.go:208-210` — Upstream guard: `if contractData.Durability != xdr.ContractDataDurabilityPersistent { return ..., false }`. This is the first check after extracting contract data.
- `internal/transform/contract_data.go:96-101` — `TransformContractData` calls `t.ContractBalanceFromContractData(ledgerEntry, passphrase)` and populates output balance fields whenever `dataBalance != nil`, with no secondary durability check.
- `internal/transform/contract_data.go:112` — `contractDataDurability` IS exported as a string field, so the durability info is present in the record, but balance field population is not conditioned on it.
- `cmd/export_ledger_entry_changes.go:217` — Production wiring passes the real `transform.ContractBalanceFromContractData` (no mock).
- `internal/transform/contract_data_test.go:61,74` — Tests use `MockContractBalanceFromContractData` and never exercise the real durability guard logic.

### Findings

The bug is confirmed. The local `ContractBalanceFromContractData` was derived from the upstream implementation but the durability guard was removed. Key evidence:

1. **Upstream has the guard**: `go-stellar-sdk` line 208-210 explicitly rejects non-persistent entries before any balance parsing.
2. **Local omits the guard**: The local function proceeds directly from data extraction to key parsing with no durability check.
3. **No compensating check elsewhere**: `TransformContractData` at line 96-101 does not re-check durability before populating balance fields.
4. **Function comment is stale**: The local function's doc comment (lines 299-305) still says it "verifies that the ledger entry corresponds to the balance entry written to contract storage by the Stellar Asset Contract," implying the same guarantees as upstream — but the durability check that enforces those guarantees is missing.
5. **Tests don't catch this**: The mock-based test strategy means the real function's missing guard is never exercised.
6. **Sibling bug already found**: The reviewed hypothesis 001 identified a *different* missing guard in the same function (native asset exclusion). This is a second independent omission in the same copy-from-upstream function.

### PoC Guidance

- **Test file**: `internal/transform/contract_data_test.go`
- **Setup**: Use `sac.BalanceToContractData(contractID, holderID, amount)` from `go-stellar-sdk/ingest/sac` to create a balance-shaped ledger entry (this creates persistent durability by default). Then manually modify the entry's `contractData.Durability` to `xdr.ContractDataDurabilityTemporary`. Create a `TransformContractDataStruct` initialized with the **real** `ContractBalanceFromContractData` (not the mock).
- **Steps**: Call `ContractBalanceFromContractData(temporaryBalanceEntry, passphrase)` directly. Then call `TransformContractData` with a change containing the temporary balance entry.
- **Assertion**: Assert that `ContractBalanceFromContractData` returns `false` (third return value) for temporary balance entries — currently it returns `true`. Assert that the transformed output has empty `ContractDataBalanceHolder` and `ContractDataBalance` fields for temporary entries — currently they are populated.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTemporaryContractBalanceExportedAsPersistent"
**Test Language**: Go

### Demonstration

The test constructs two identical balance-shaped `ContractDataEntry` objects — one with `ContractDataDurabilityPersistent` and one with `ContractDataDurabilityTemporary`. Both are passed to the real `ContractBalanceFromContractData` function. The persistent entry correctly returns a balance (baseline), and the temporary entry also returns `ok=true` with `amount=1000000` — proving the function accepts temporary entries that should be rejected. The upstream SDK's durability guard at `contract_data.go:208-210` would reject this entry immediately.

### Test Body

```go
func TestTemporaryContractBalanceExportedAsPersistent(t *testing.T) {
	passphrase := "Test SDF Network ; September 2015"

	// Build a contract ID for a non-native SAC (to avoid the native exclusion path)
	issuerKP := keypair.MustRandom()
	asset, err := xdr.NewCreditAsset("TEST", issuerKP.Address())
	if err != nil {
		t.Fatalf("failed to create test asset: %v", err)
	}
	contractID, err := asset.ContractID(passphrase)
	if err != nil {
		t.Fatalf("failed to get contract ID: %v", err)
	}

	// Build a holder contract address
	holderID := xdr.Hash{0x01, 0x02, 0x03, 0x04}
	holderContractID := xdr.ContractId(holderID)
	holderAddress := xdr.ScAddress{
		Type:       xdr.ScAddressTypeScAddressTypeContract,
		ContractId: &holderContractID,
	}

	// Build a balance-shaped key: ["Balance", ScAddress(holder)]
	balanceSym := xdr.ScSymbol("Balance")
	keyVec := xdr.ScVec{
		xdr.ScVal{Type: xdr.ScValTypeScvSymbol, Sym: &balanceSym},
		xdr.ScVal{Type: xdr.ScValTypeScvAddress, Address: &holderAddress},
	}
	keyVecPtr := &keyVec

	// Build a balance-shaped value: {amount: i128, authorized: bool, clawback: bool}
	amountSym := xdr.ScSymbol("amount")
	authorizedSym := xdr.ScSymbol("authorized")
	clawbackSym := xdr.ScSymbol("clawback")
	trueVal := true
	balanceMap := xdr.ScMap{
		{
			Key: xdr.ScVal{Type: xdr.ScValTypeScvSymbol, Sym: &amountSym},
			Val: xdr.ScVal{
				Type: xdr.ScValTypeScvI128,
				I128: &xdr.Int128Parts{Hi: 0, Lo: 1000000},
			},
		},
		{
			Key: xdr.ScVal{Type: xdr.ScValTypeScvSymbol, Sym: &authorizedSym},
			Val: xdr.ScVal{Type: xdr.ScValTypeScvBool, B: &trueVal},
		},
		{
			Key: xdr.ScVal{Type: xdr.ScValTypeScvSymbol, Sym: &clawbackSym},
			Val: xdr.ScVal{Type: xdr.ScValTypeScvBool, B: &trueVal},
		},
	}
	balanceMapPtr := &balanceMap

	sacContractID := xdr.ContractId(contractID)

	// ---- Test 1: Persistent entry returns balance (baseline) ----
	persistentEntry := xdr.LedgerEntry{
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeContractData,
			ContractData: &xdr.ContractDataEntry{
				Contract: xdr.ScAddress{
					Type:       xdr.ScAddressTypeScAddressTypeContract,
					ContractId: &sacContractID,
				},
				Key:        xdr.ScVal{Type: xdr.ScValTypeScvVec, Vec: &keyVecPtr},
				Durability: xdr.ContractDataDurabilityPersistent,
				Val:        xdr.ScVal{Type: xdr.ScValTypeScvMap, Map: &balanceMapPtr},
			},
		},
	}

	holder, amt, ok := ContractBalanceFromContractData(persistentEntry, passphrase)
	if !ok {
		t.Fatal("Baseline failed: persistent balance entry was not recognized")
	}
	if amt.Cmp(big.NewInt(1000000)) != 0 {
		t.Fatalf("Baseline failed: expected amount 1000000, got %s", amt.String())
	}
	if holder != holderID {
		t.Fatalf("Baseline failed: holder mismatch")
	}
	t.Logf("Baseline: persistent entry correctly returns balance (holder=%x, amount=%s)", holder[:4], amt)

	// ---- Test 2: Temporary entry should NOT return balance ----
	temporaryEntry := xdr.LedgerEntry{
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeContractData,
			ContractData: &xdr.ContractDataEntry{
				Contract: xdr.ScAddress{
					Type:       xdr.ScAddressTypeScAddressTypeContract,
					ContractId: &sacContractID,
				},
				Key:        xdr.ScVal{Type: xdr.ScValTypeScvVec, Vec: &keyVecPtr},
				Durability: xdr.ContractDataDurabilityTemporary,
				Val:        xdr.ScVal{Type: xdr.ScValTypeScvMap, Map: &balanceMapPtr},
			},
		},
	}

	_, tempAmt, tempOk := ContractBalanceFromContractData(temporaryEntry, passphrase)

	// BUG: The function returns true + a balance for temporary entries.
	// It should return false because temporary storage is not durable token state.
	if tempOk && tempAmt != nil {
		t.Errorf("BUG CONFIRMED: ContractBalanceFromContractData returned ok=true, amount=%s "+
			"for a TEMPORARY contract data entry. Upstream SDK rejects non-persistent entries.",
			tempAmt.String())
	} else {
		t.Log("No bug: temporary entry was correctly rejected")
	}
}
```

### Test Output

```
=== RUN   TestTemporaryContractBalanceExportedAsPersistent
    data_integrity_poc_test.go:159: Baseline: persistent entry correctly returns balance (holder=01020304, amount=1000000)
    data_integrity_poc_test.go:182: BUG CONFIRMED: ContractBalanceFromContractData returned ok=true, amount=1000000 for a TEMPORARY contract data entry. Upstream SDK rejects non-persistent entries.
--- FAIL: TestTemporaryContractBalanceExportedAsPersistent (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.775s
```
