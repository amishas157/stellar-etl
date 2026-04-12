# 031: Extend/restore footprint exports first footprint contract

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports singular `contract_id` and `contract_code_hash` metadata for `extend_footprint_ttl` and `restore_footprint` by scanning the Soroban footprint and returning the first matching entry. These operations target a footprint set, not a single contract, so valid multi-contract footprints produce plausible but wrong scalar metadata that changes when footprint order changes.

The bug is reproducible through the production transform path for both operation types. Swapping two `ContractData` keys in the same footprint flips the exported `contract_id` while `ledger_key_hash` still reports both keys, proving the singleton metadata is an arbitrary projection of set-valued state.

## Root Cause

Both TTL-operation branches in `extractOperationDetails()` and `transactionOperationWrapper.Details()` call `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()`. Those helpers iterate the Soroban footprint arrays and return the first `ContractData` or `ContractCode` match they find, with no logic to determine whether a singular contract identifier is semantically valid for the operation.

## Reproduction

Create an `extend_footprint_ttl` or `restore_footprint` operation whose Soroban footprint contains `ContractData` keys for two different contracts. Run the operation through `TransformOperation()`, then swap the `ReadWrite` order of those two keys and run it again.

The exported `contract_id` flips to whichever contract appears first in the footprint, while `ledger_key_hash` still contains both footprint entries. The exported row therefore contains a syntactically valid but order-dependent contract identifier that does not represent the operation as a whole.

## Affected Code

- `internal/transform/operation.go:1144-1159` — `extractOperationDetails()` assigns singular `contract_id` and `contract_code_hash` to both TTL operations
- `internal/transform/operation.go:1766-1781` — `transactionOperationWrapper.Details()` duplicates the same mapping for the parallel details path
- `internal/transform/operation.go:1808-1856` — `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` return the first matching footprint entry

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestExtendFootprintTtlArbitraryContractId`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestExtendFootprintTtlArbitraryContractId -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestExtendFootprintTtlArbitraryContractId(t *testing.T) {
	contractHashA := xdr.Hash([32]byte{
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
	})
	contractHashB := xdr.Hash([32]byte{
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
	})
	contractIDA := xdr.ContractId(contractHashA)
	contractIDB := xdr.ContractId(contractHashB)

	contractAddressA := strkey.MustEncode(strkey.VersionByteContract, contractHashA[:])
	contractAddressB := strkey.MustEncode(strkey.VersionByteContract, contractHashB[:])

	makeContractDataKey := func(contractID xdr.ContractId) xdr.LedgerKey {
		return xdr.LedgerKey{
			Type: xdr.LedgerEntryTypeContractData,
			ContractData: &xdr.LedgerKeyContractData{
				Contract: xdr.ScAddress{
					Type:       xdr.ScAddressTypeScAddressTypeContract,
					ContractId: &contractID,
				},
				Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
				Durability: xdr.ContractDataDurabilityPersistent,
			},
		}
	}

	buildOperation := func(operationType xdr.OperationType) xdr.Operation {
		operation := xdr.Operation{
			SourceAccount: &genericSourceAccount,
			Body: xdr.OperationBody{
				Type: operationType,
			},
		}
		switch operationType {
		case xdr.OperationTypeExtendFootprintTtl:
			operation.Body.ExtendFootprintTtlOp = &xdr.ExtendFootprintTtlOp{
				Ext:      xdr.ExtensionPoint{V: 0},
				ExtendTo: 5000,
			}
		case xdr.OperationTypeRestoreFootprint:
			operation.Body.RestoreFootprintOp = &xdr.RestoreFootprintOp{
				Ext: xdr.ExtensionPoint{V: 0},
			}
		default:
			t.Fatalf("unexpected operation type %s", operationType)
		}
		return operation
	}

	buildTransaction := func(operation xdr.Operation, readWrite []xdr.LedgerKey) ingest.LedgerTransaction {
		envelope := xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: genericSourceAccount,
				Memo:          xdr.Memo{},
				Operations:    []xdr.Operation{operation},
				Ext: xdr.TransactionExt{
					V: 1,
					SorobanData: &xdr.SorobanTransactionData{
						Ext: xdr.SorobanTransactionDataExt{V: 0},
						Resources: xdr.SorobanResources{
							Footprint: xdr.LedgerFootprint{
								ReadWrite: readWrite,
								ReadOnly:  []xdr.LedgerKey{},
							},
						},
						ResourceFee: 100,
					},
				},
			},
		}

		tx := genericLedgerTransaction
		tx.Envelope = xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		}
		return tx
	}

	runTransform := func(operation xdr.Operation, tx ingest.LedgerTransaction) (string, []string) {
		output, err := TransformOperation(operation, 0, tx, 0, makeLedgerCloseMeta(), "test")
		if err != nil {
			t.Fatalf("TransformOperation failed: %v", err)
		}

		contractID, ok := output.OperationDetails["contract_id"].(string)
		if !ok {
			t.Fatalf("contract_id type = %T, want string", output.OperationDetails["contract_id"])
		}

		ledgerKeyHashes, ok := output.OperationDetails["ledger_key_hash"].([]string)
		if !ok {
			t.Fatalf("ledger_key_hash type = %T, want []string", output.OperationDetails["ledger_key_hash"])
		}

		return contractID, ledgerKeyHashes
	}

	for _, operationType := range []xdr.OperationType{
		xdr.OperationTypeExtendFootprintTtl,
		xdr.OperationTypeRestoreFootprint,
	} {
		t.Run(operationType.String(), func(t *testing.T) {
			operation := buildOperation(operationType)

			firstOrderContractID, firstOrderKeyHashes := runTransform(operation, buildTransaction(operation, []xdr.LedgerKey{
				makeContractDataKey(contractIDB),
				makeContractDataKey(contractIDA),
			}))

			secondOrderContractID, secondOrderKeyHashes := runTransform(operation, buildTransaction(operation, []xdr.LedgerKey{
				makeContractDataKey(contractIDA),
				makeContractDataKey(contractIDB),
			}))

			if firstOrderContractID != contractAddressB {
				t.Fatalf("first ordering exported %q, want first footprint contract %q", firstOrderContractID, contractAddressB)
			}

			if secondOrderContractID != contractAddressA {
				t.Fatalf("second ordering exported %q, want first footprint contract %q", secondOrderContractID, contractAddressA)
			}

			if firstOrderContractID == secondOrderContractID {
				t.Fatalf("contract_id stayed %q after swapping footprint order", firstOrderContractID)
			}

			if len(firstOrderKeyHashes) != 2 || len(secondOrderKeyHashes) != 2 {
				t.Fatalf("ledger_key_hash lengths = %d and %d, want 2 entries for both footprints", len(firstOrderKeyHashes), len(secondOrderKeyHashes))
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `extend_footprint_ttl` and `restore_footprint` should not export a singular authoritative `contract_id` or `contract_code_hash` unless the footprint unambiguously identifies a single contract; otherwise the scalar fields should remain empty or be replaced with set-valued metadata.
- **Actual**: the exporter returns the first matching footprint contract or code entry, so the scalar metadata changes when the same footprint set is reordered.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC runs `TransformOperation()` and proves both TTL operation types export different `contract_id` values when only footprint order changes.
2. Realistic preconditions: YES — Soroban TTL operations carry only a footprint and valid footprints can contain entries for multiple contracts.
3. Bug vs by-design: BUG — the XDR operations have no singular contract target, and there is no documented justification for exporting an order-dependent singleton as authoritative metadata.
4. Final severity: High — this corrupts structural contract metadata without directly changing monetary values.
5. In scope: YES — it is a concrete data-correctness issue in `internal/transform/`.
6. Test correctness: CORRECT — the test uses production helpers, real XDR operation types, and demonstrates output variance caused solely by footprint ordering.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Stop populating singular `contract_id` and `contract_code_hash` for TTL operations from first-match footprint scans. Either export only the set-valued footprint metadata already present in `ledger_key_hash`, or add explicit multi-value contract/code fields and leave the legacy scalar fields empty when more than one matching contract or code entry exists.
