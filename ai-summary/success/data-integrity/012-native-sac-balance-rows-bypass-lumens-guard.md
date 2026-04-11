# 012: Native SAC balance rows bypass lumens guard

**Date**: 2026-04-11
**Severity**: High
**Impact**: Structural data corruption
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformContractData` exports native-asset SAC balance rows with populated `balance_holder` and `balance` columns, even though the upstream SAC parser explicitly rejects lumens asset stats. The local helper computes the native asset contract ID but never compares it to the row's contract ID, so unsupported native rows are serialized as normal `soroban_contract_data` balance facts.

## Root Cause

`ContractBalanceFromContractData` is a fork of the SDK SAC helper, but its lumens exclusion check was dropped. The function still calls `xdr.MustNewNativeAsset().ContractID(passphrase)` and discards the result, then `TransformContractData` copies any non-nil parsed balance into exported `balance_*` fields.

## Reproduction

Build a real native SAC balance `ContractData` entry under the contract ID returned by `xdr.MustNewNativeAsset().ContractID(passphrase)`, then feed it through `TransformContractData`. With the local helper, the output row contains a non-empty contract holder and `balance="1000000"`; with the upstream SDK helper wired into the same transform as a control, those fields stay empty.

## Affected Code

- `internal/transform/contract_data.go:96-100` — `TransformContractData` populates `contract_data_balance_holder` and `contract_data_balance` whenever the helper returns a balance.
- `internal/transform/contract_data.go:306-315` — `ContractBalanceFromContractData` computes the native asset contract ID and discards it instead of rejecting lumens rows.
- `internal/transform/contract_data.go:317-378` — the rest of the helper accepts the native SAC balance row and returns a holder/balance pair that looks valid.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestNativeSACBalanceBypassesLumensGuard`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, add `github.com/stellar/go-stellar-sdk/ingest` and `github.com/stellar/go-stellar-sdk/ingest/sac` to that file's imports, then run `go test ./internal/transform/... -run TestNativeSACBalanceBypassesLumensGuard -v`.

### Test Body

```go
func TestNativeSACBalanceBypassesLumensGuard(t *testing.T) {
	const passphrase = "Test SDF Network ; September 2015"

	nativeContractID, err := xdr.MustNewNativeAsset().ContractID(passphrase)
	if err != nil {
		t.Fatalf("failed to compute native asset contract ID: %v", err)
	}

	holderID := [32]byte{0xAA, 0xBB, 0xCC}
	amount := uint64(1_000_000)
	ledgerEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data:                  sac.BalanceToContractData(nativeContractID, holderID, amount),
	}

	_, upstreamBalance, upstreamOK := sac.ContractBalanceFromContractData(ledgerEntry, passphrase)
	if upstreamOK || upstreamBalance != nil {
		t.Fatalf("test setup invalid: upstream SAC helper should reject native/lumens balance rows; got ok=%v balance=%v", upstreamOK, upstreamBalance)
	}

	localHolder, localBalance, localOK := ContractBalanceFromContractData(ledgerEntry, passphrase)
	if !localOK || localBalance == nil {
		t.Fatalf("expected local helper to currently accept native balance rows; got ok=%v balance=%v", localOK, localBalance)
	}
	if localHolder != holderID {
		t.Fatalf("local helper returned unexpected holder: got %x want %x", localHolder, holderID)
	}
	if localBalance.String() != "1000000" {
		t.Fatalf("local helper returned unexpected balance: got %s want 1000000", localBalance.String())
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeContractData,
		Post:       &ledgerEntry,
	}
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
		},
	}

	upstreamTransform := NewTransformContractDataStruct(AssetFromContractData, sac.ContractBalanceFromContractData)
	upstreamOutput, err, ok := upstreamTransform.TransformContractData(change, passphrase, header)
	if err != nil || !ok {
		t.Fatalf("upstream control transform failed: ok=%v err=%v", ok, err)
	}
	if upstreamOutput.ContractDataBalanceHolder != "" || upstreamOutput.ContractDataBalance != "" {
		t.Fatalf("test setup invalid: upstream control transform exported native balance fields: holder=%q balance=%q", upstreamOutput.ContractDataBalanceHolder, upstreamOutput.ContractDataBalance)
	}

	transformer := NewTransformContractDataStruct(AssetFromContractData, ContractBalanceFromContractData)
	output, err, ok := transformer.TransformContractData(change, passphrase, header)
	if err != nil || !ok {
		t.Fatalf("local transform failed: ok=%v err=%v", ok, err)
	}

	if output.ContractDataBalanceHolder != "" || output.ContractDataBalance != "" {
		t.Fatalf("BUG CONFIRMED: native SAC balance row exported balance fields: holder=%q balance=%q; expected both fields to stay empty because lumens rows are unsupported", output.ContractDataBalanceHolder, output.ContractDataBalance)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: native/lumens SAC balance rows should be rejected before balance extraction, so exported `balance_holder` and `balance` stay empty.
- **Actual**: the local helper accepts the native row and `TransformContractData` emits populated `balance_holder` and `balance` fields.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC builds a real native SAC balance entry, calls the production helper, and runs the production transform path that writes the exported `balance_*` fields.
2. Realistic preconditions: YES — native SAC balance entries are standard contract-data rows for the native asset contract; the test uses the SDK's SAC helper only to construct the exact XDR shape.
3. Bug vs by-design: BUG — the local helper is visibly forked from the SDK version, still computes the native contract ID, and lacks any comment or test justifying a deliberate lumens-export divergence.
4. Final severity: High — the export emits plausible but unsupported balance facts in `soroban_contract_data`, corrupting downstream interpretation of contract-data balance rows without changing raw arithmetic.
5. In scope: YES — this is a concrete wrong-output path in the public contract-data export flow.
6. Test correctness: CORRECT — the test uses the upstream helper and an upstream-wired transform as controls, so the failure is tied to the local helper's missing lumens guard rather than bad fixture setup.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Restore the upstream lumens exclusion in `ContractBalanceFromContractData`: keep the native asset contract ID returned by `xdr.MustNewNativeAsset().ContractID(passphrase)` and return `false` when the current row's contract ID matches it. Ideally sync the helper's whole preamble with the SDK version so future guard changes do not drift again.
