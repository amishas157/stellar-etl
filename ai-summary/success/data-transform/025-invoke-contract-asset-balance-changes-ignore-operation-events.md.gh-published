# 025: Invoke-contract asset balance changes ignore operation events

**Date**: 2026-04-11
**Severity**: High
**Impact**: operation detail corruption
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`invoke_contract` operation export populates `details.asset_balance_changes` from `GetDiagnosticEvents()` instead of the canonical operation contract-event stream. When txmeta is produced without duplicating contract events into diagnostics, the ETL silently exports an empty balance-change list even though the SAC transfer/mint/burn/clawback event is present in metadata.

This is configuration-dependent rather than hypothetical: the CLI accepts arbitrary `--core-config`, and the SDK explicitly documents that diagnostic events may include contract events only conditionally. The result is silent, plausible-looking wrong output in `history_operations.details`.

## Root Cause

`extractOperationDetails()` calls `parseAssetBalanceChangesFromContractEvents()` for `invoke_contract` operations, but that helper reads only `transaction.GetDiagnosticEvents()`. The canonical contract events live in `GetContractEvents()`/`GetTransactionEvents().OperationEvents`, so valid SAC events are dropped whenever they are not mirrored into the diagnostic-event slice.

## Reproduction

Create a Soroban `ingest.LedgerTransaction` whose V3 `SorobanMeta.Events` contains a valid SAC transfer event while `SorobanMeta.DiagnosticEvents` is empty. The production helper returns `[]` for `asset_balance_changes`, but `GetContractEvents()` still returns the event, proving the ETL ignored available source data and made output depend on txmeta generation settings.

## Affected Code

- `internal/transform/operation.go:extractOperationDetails:1063-1097` — `invoke_contract` details always source `asset_balance_changes` through the diagnostic-only helper.
- `internal/transform/operation.go:parseAssetBalanceChangesFromContractEvents:1942-1975` — standalone helper reads `transaction.GetDiagnosticEvents()` and never inspects operation contract events.
- `internal/transform/operation.go:(*transactionOperationWrapper).parseAssetBalanceChangesFromContractEvents:1907-1940` — sibling path has the same diagnostic-only dependency.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestInvokeContractAssetBalanceChangesMissingWhenDiagnosticEventsEmpty`
- **Test language**: `go`
- **How to run**: `go test ./internal/transform/... -run TestInvokeContractAssetBalanceChangesMissingWhenDiagnosticEventsEmpty -v`

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/network"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestInvokeContractAssetBalanceChangesMissingWhenDiagnosticEventsEmpty demonstrates
// that parseAssetBalanceChangesFromContractEvents returns an empty result when contract
// events exist in SorobanMeta.Events but NOT in SorobanMeta.DiagnosticEvents.
// This simulates a deployment where ENABLE_SOROBAN_DIAGNOSTIC_EVENTS=false.
func TestInvokeContractAssetBalanceChangesMissingWhenDiagnosticEventsEmpty(t *testing.T) {
	passphrase := network.TestNetworkPassphrase

	// Build a native asset and compute its SAC contract ID.
	nativeAsset := xdr.Asset{Type: xdr.AssetTypeAssetTypeNative}
	contractIDBytes, err := nativeAsset.ContractID(passphrase)
	if err != nil {
		t.Fatalf("failed to compute contract ID: %v", err)
	}
	contractID := xdr.ContractId(contractIDBytes)

	// Build a SAC transfer ContractEvent.
	fromAddr := xdr.ScAddress{
		Type:      xdr.ScAddressTypeScAddressTypeAccount,
		AccountId: &testAccount1ID,
	}
	toAddr := xdr.ScAddress{
		Type:      xdr.ScAddressTypeScAddressTypeAccount,
		AccountId: &testAccount2ID,
	}
	transferSym := xdr.ScSymbol("transfer")
	assetStr := xdr.ScString("native")
	amount := xdr.Int128Parts{Hi: 0, Lo: 1000000}

	sacEvent := xdr.ContractEvent{
		Type:       xdr.ContractEventTypeContract,
		ContractId: &contractID,
		Body: xdr.ContractEventBody{
			V: 0,
			V0: &xdr.ContractEventV0{
				Topics: xdr.ScVec{
					{Type: xdr.ScValTypeScvSymbol, Sym: &transferSym},
					{Type: xdr.ScValTypeScvAddress, Address: &fromAddr},
					{Type: xdr.ScValTypeScvAddress, Address: &toAddr},
					{Type: xdr.ScValTypeScvString, Str: &assetStr},
				},
				Data: xdr.ScVal{
					Type: xdr.ScValTypeScvI128,
					I128: &xdr.Int128Parts{
						Hi: amount.Hi,
						Lo: amount.Lo,
					},
				},
			},
		},
	}

	// Construct V3 TxMeta with the contract event in SorobanMeta.Events but not
	// in DiagnosticEvents, simulating a txmeta producer that does not duplicate
	// contract events into the diagnostic stream.
	sorobanMeta := &xdr.SorobanTransactionMeta{
		Events:           []xdr.ContractEvent{sacEvent},
		DiagnosticEvents: []xdr.DiagnosticEvent{},
		ReturnValue:      xdr.ScVal{Type: xdr.ScValTypeScvVoid},
	}

	txMeta := xdr.TransactionMeta{
		V: 3,
		V3: &xdr.TransactionMetaV3{
			SorobanMeta: sorobanMeta,
		},
	}

	sorobanData := xdr.SorobanTransactionData{
		Resources: xdr.SorobanResources{
			Footprint: xdr.LedgerFootprint{},
		},
		ResourceFee: 100,
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type: xdr.OperationTypeInvokeHostFunction,
								InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
									HostFunction: xdr.HostFunction{
										Type: xdr.HostFunctionTypeHostFunctionTypeInvokeContract,
										InvokeContract: &xdr.InvokeContractArgs{
											ContractAddress: xdr.ScAddress{
												Type:       xdr.ScAddressTypeScAddressTypeContract,
												ContractId: &contractID,
											},
											FunctionName: "transfer",
										},
									},
								},
							},
						},
					},
					Ext: xdr.TransactionExt{
						V:           1,
						SorobanData: &sorobanData,
					},
				},
			},
		},
		UnsafeMeta: txMeta,
	}

	balanceChanges, err := parseAssetBalanceChangesFromContractEvents(tx, passphrase)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(balanceChanges) != 0 {
		t.Fatalf("expected 0 balance changes from diagnostic-events path, got %d", len(balanceChanges))
	}

	contractEvents, err := tx.GetContractEvents()
	if err != nil {
		t.Fatalf("unexpected error from GetContractEvents: %v", err)
	}
	if len(contractEvents) != 1 {
		t.Fatalf("expected 1 contract event from GetContractEvents, got %d", len(contractEvents))
	}

	sorobanMeta.DiagnosticEvents = []xdr.DiagnosticEvent{
		{
			InSuccessfulContractCall: true,
			Event:                    sacEvent,
		},
	}
	tx.UnsafeMeta.V3.SorobanMeta = sorobanMeta

	balanceChangesWithDiag, err := parseAssetBalanceChangesFromContractEvents(tx, passphrase)
	if err != nil {
		t.Fatalf("unexpected error with diagnostic events populated: %v", err)
	}
	if len(balanceChangesWithDiag) != 1 {
		t.Fatalf("expected 1 balance change when diagnostic events are populated, got %d", len(balanceChangesWithDiag))
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `asset_balance_changes` should be derived from the operation's canonical SAC contract events, so valid balance-change rows appear regardless of diagnostic-event duplication.
- **Actual**: `asset_balance_changes` is empty unless contract events are also present in `DiagnosticEvents`.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production helper on a real `ingest.LedgerTransaction` and demonstrates that canonical SAC events are ignored when diagnostics are empty.
2. Realistic preconditions: YES — the CLI accepts user-supplied `--core-config`, and the SDK documents that diagnostic-event duplication is conditional.
3. Bug vs by-design: BUG — sibling code already uses `GetTransactionEvents()`, and the helper's stated purpose is to parse contract events, not optional diagnostics.
4. Final severity: High — this silently drops non-empty `asset_balance_changes` rows from valid `invoke_contract` exports, corrupting operation detail output.
5. In scope: YES — this is silent exported data corruption in the transform layer.
6. Test correctness: CORRECT — it proves the data exists in canonical metadata, proves the helper misses it, and proves the helper starts working when only the diagnostic duplication changes.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Populate balance changes from the canonical contract-event stream instead of the diagnostic-event stream. In `extractOperationDetails()` and the wrapper method, use `GetTransactionEvents().OperationEvents[operationIndex]` (or `GetContractEventsForOperation`) and only consume diagnostics for truly diagnostic-only events.
