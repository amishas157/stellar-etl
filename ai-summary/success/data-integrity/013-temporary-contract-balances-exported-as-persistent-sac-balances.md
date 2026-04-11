# 013: Temporary contract balances exported as persistent SAC balances

**Date**: 2026-04-11
**Severity**: High
**Impact**: Structural data corruption
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformContractData` exports populated `balance_holder` and `balance` columns for temporary `ContractData` rows whose key/value shape matches a Stellar Asset Contract balance entry. The local `ContractBalanceFromContractData` fork accepts those temporary rows even though the upstream SAC parser rejects non-persistent durability before parsing any balance payload.

## Root Cause

`ContractBalanceFromContractData` was forked from the SDK SAC helper but lost the `contractData.Durability != xdr.ContractDataDurabilityPersistent` guard near the top of the function. `TransformContractData` treats any non-nil parsed balance as exportable and copies it into the output row without a secondary durability check.

## Reproduction

Create a real SAC balance-shaped `ContractData` entry with `sac.BalanceToContractData`, then flip its durability from `Persistent` to `Temporary`. Feeding that entry through the production `TransformContractData` path produces a row whose `contract_durability` is `ContractDataDurabilityTemporary` but whose `balance_holder` and `balance` columns are still populated, so downstream consumers see an ephemeral storage row as a durable token balance.

## Affected Code

- `internal/transform/contract_data.go:96-100` — `TransformContractData` copies any parsed holder/amount into exported `balance_*` fields.
- `internal/transform/contract_data.go:306-315` — `ContractBalanceFromContractData` extracts `ContractData` but omits the upstream persistent-durability guard.
- `internal/transform/contract_data.go:321-378` — the helper accepts a balance-shaped temporary row and returns a holder/amount pair.
- `cmd/export_ledger_entry_changes.go:212-218` — production export wiring uses the real local helper for every contract-data change.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTemporaryContractBalanceExportedAsPersistent`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, add `github.com/stellar/go-stellar-sdk/ingest` and `github.com/stellar/go-stellar-sdk/ingest/sac` to that file's imports, then run `go test ./internal/transform/... -run TestTemporaryContractBalanceExportedAsPersistent -v`.

### Test Body

```go
func TestTemporaryContractBalanceExportedAsPersistent(t *testing.T) {
	passphrase := "Test SDF Network ; September 2015"
	const amount = uint64(1000000)

	persistentEntry := makeSACBalanceLedgerEntry(t, passphrase, xdr.ContractDataDurabilityPersistent, amount)
	temporaryEntry := makeSACBalanceLedgerEntry(t, passphrase, xdr.ContractDataDurabilityTemporary, amount)

	holder, persistentAmount, persistentOK := ContractBalanceFromContractData(persistentEntry, passphrase)
	if !persistentOK || persistentAmount == nil {
		t.Fatal("baseline failed: persistent SAC balance entry was not recognized by the local parser")
	}
	if persistentAmount.Uint64() != amount {
		t.Fatalf("baseline failed: expected persistent amount %d, got %s", amount, persistentAmount.String())
	}

	upstreamHolder, upstreamAmount, upstreamOK := sac.ContractBalanceFromContractData(persistentEntry, passphrase)
	if !upstreamOK || upstreamAmount == nil {
		t.Fatal("baseline failed: persistent SAC balance entry was not recognized by the upstream parser")
	}
	if upstreamHolder != holder || upstreamAmount.Cmp(persistentAmount) != 0 {
		t.Fatalf("baseline failed: local and upstream parsers disagreed for persistent balance entry")
	}

	_, temporaryAmount, temporaryOK := ContractBalanceFromContractData(temporaryEntry, passphrase)
	if temporaryOK || temporaryAmount != nil {
		t.Errorf("temporary ContractData should be rejected as a SAC balance, got ok=%v amount=%v", temporaryOK, temporaryAmount)
	}

	if _, _, upstreamTemporaryOK := sac.ContractBalanceFromContractData(temporaryEntry, passphrase); upstreamTemporaryOK {
		t.Fatal("sanity check failed: upstream parser unexpectedly accepted temporary ContractData as a SAC balance")
	}

	transformer := NewTransformContractDataStruct(AssetFromContractData, ContractBalanceFromContractData)
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue:  xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	persistentOutput, err, ok := transformer.TransformContractData(
		ingest.Change{
			ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
			Type:       xdr.LedgerEntryTypeContractData,
			Post:       &persistentEntry,
		},
		passphrase,
		header,
	)
	if err != nil || !ok {
		t.Fatalf("baseline failed: persistent contract data transform returned err=%v ok=%v", err, ok)
	}
	if persistentOutput.ContractDataBalance != "1000000" || persistentOutput.ContractDataBalanceHolder == "" {
		t.Fatalf("baseline failed: persistent contract data should export balance fields, got holder=%q balance=%q",
			persistentOutput.ContractDataBalanceHolder, persistentOutput.ContractDataBalance)
	}

	temporaryOutput, err, ok := transformer.TransformContractData(
		ingest.Change{
			ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
			Type:       xdr.LedgerEntryTypeContractData,
			Post:       &temporaryEntry,
		},
		passphrase,
		header,
	)
	if err != nil || !ok {
		t.Fatalf("temporary contract data transform returned err=%v ok=%v", err, ok)
	}
	if temporaryOutput.ContractDurability != xdr.ContractDataDurabilityTemporary.String() {
		t.Fatalf("expected temporary output durability, got %q", temporaryOutput.ContractDurability)
	}
	if temporaryOutput.ContractDataBalanceHolder != "" || temporaryOutput.ContractDataBalance != "" {
		t.Errorf("temporary ContractData should not populate exported balance fields, got holder=%q balance=%q",
			temporaryOutput.ContractDataBalanceHolder, temporaryOutput.ContractDataBalance)
	}
}

func makeSACBalanceLedgerEntry(t *testing.T, passphrase string, durability xdr.ContractDataDurability, amount uint64) xdr.LedgerEntry {
	t.Helper()

	asset, err := xdr.NewCreditAsset("TEST", "GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA")
	if err != nil {
		t.Fatalf("failed to create test asset: %v", err)
	}
	contractID, err := asset.ContractID(passphrase)
	if err != nil {
		t.Fatalf("failed to derive asset contract ID: %v", err)
	}

	var holderID [32]byte
	holderID[0] = 0x01
	holderID[1] = 0x02
	holderID[2] = 0x03
	holderID[3] = 0x04

	entry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 24229503,
		Data:                  sac.BalanceToContractData(contractID, holderID, amount),
	}
	entry.Data.ContractData.Durability = durability
	return entry
}
```

## Expected vs Actual Behavior

- **Expected**: temporary `ContractData` rows should never be treated as persistent SAC balances, so the helper should reject them and exported `balance_holder` / `balance` fields should stay empty.
- **Actual**: the local helper accepts the temporary row and `TransformContractData` exports populated `balance_holder` and `balance` values even though `contract_durability` is `Temporary`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC constructs a real SAC balance-shaped ledger entry, changes only durability to `Temporary`, and runs the production helper plus the production `TransformContractData` export path.
2. Realistic preconditions: YES — temporary Soroban `ContractData` rows are a normal ledger construct, and the test uses the SDK SAC constructor only to build the exact on-chain XDR shape.
3. Bug vs by-design: BUG — the local helper is a visible fork of the upstream SAC parser, its doc comment still claims SAC-balance verification semantics, and there is no local documentation or guard justifying acceptance of temporary rows as exported balances.
4. Final severity: High — this is structural misclassification of ephemeral storage into durable-looking `balance_*` export columns; the values are plausible and silently wrong, but the corruption is semantic mapping rather than numeric truncation.
5. In scope: YES — the public contract-data export path emits a concrete wrong row without crashing or logging an error.
6. Test correctness: CORRECT — the PoC uses persistent rows as a baseline and the upstream SAC helper as a control, so the failing assertions isolate the local missing durability guard rather than bad fixture setup.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Restore the upstream durability precondition in `ContractBalanceFromContractData` by returning `false` when `contractData.Durability != xdr.ContractDataDurabilityPersistent`. Keeping this helper aligned with the upstream SAC implementation would also reduce future guard drift in the copied parser.
