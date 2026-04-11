# 024: Create-contract uses first footprint contract

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports `contract_id` for `create_contract` and `create_contract_v2` by scanning the Soroban footprint and returning the first `ContractData` entry it sees. The contract being created is actually determined by the network passphrase plus `ContractIdPreimage`, so a transaction whose footprint includes another contract first produces a plausible but wrong exported contract ID.

The issue is not limited to malformed tests: a valid create-contract transaction can carry multiple `ContractData` keys, and the current transformer never checks that the footprint match corresponds to the newly created contract instance.

## Root Cause

Both create-contract branches call `contractIdFromTxEnvelope()` instead of deriving the contract ID from `args.ContractIdPreimage`. That helper walks `ReadWrite` and then `ReadOnly`, returning the first `ContractData` contract ID it encounters, while `contractIdFromContractData()` accepts any `ContractData` key without verifying that it is the new contract instance for this operation.

## Reproduction

Create a Soroban `create_contract` operation whose `ContractIdPreimage` is `FROM_ADDRESS` for a deployer contract. Build the transaction footprint so the deployer's `ContractData` entry appears before the created contract's `ContractInstance` entry, then run the operation through `TransformOperation()`.

The exported row contains a non-empty `contract_id`, but it is the deployer contract's ID rather than the created contract ID derived from the network passphrase and preimage. Downstream consumers receive a syntactically valid contract ID string tied to the wrong contract.

## Affected Code

- `internal/transform/operation.go:1099-1127` — `extractOperationDetails()` assigns `contract_id` for `create_contract` and `create_contract_v2` from the footprint helper
- `internal/transform/operation.go:1721-1749` — `transactionOperationWrapper.Details()` duplicates the same order-dependent mapping
- `internal/transform/operation.go:1808-1838` — `contractIdFromTxEnvelope()` / `contractIdFromContractData()` return the first footprint contract ID without checking it matches the created contract

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestCreateContractUsesFirstFootprintContract`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestCreateContractUsesFirstFootprintContract -v`.

### Test Body

```go
package transform

import (
	"crypto/sha256"
	"testing"

	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func deriveContractIDString(t *testing.T, passphrase string, preimage xdr.ContractIdPreimage) string {
	t.Helper()

	networkID := xdr.Hash(sha256.Sum256([]byte(passphrase)))
	hashPreimage := xdr.HashIdPreimage{
		Type: xdr.EnvelopeTypeEnvelopeTypeContractId,
		ContractId: &xdr.HashIdPreimageContractId{
			NetworkId:          networkID,
			ContractIdPreimage: preimage,
		},
	}

	rawPreimage, err := hashPreimage.MarshalBinary()
	if err != nil {
		t.Fatalf("marshal contract-id preimage: %v", err)
	}

	contractHash := sha256.Sum256(rawPreimage)
	return strkey.MustEncode(strkey.VersionByteContract, contractHash[:])
}

func TestCreateContractUsesFirstFootprintContract(t *testing.T) {
	const networkPassphrase = "Test SDF Network ; September 2015"

	deployerHash := xdr.Hash{}
	deployerContract := xdr.ContractId(deployerHash)
	deployerContractID := strkey.MustEncode(strkey.VersionByteContract, deployerHash[:])

	preimage := xdr.ContractIdPreimage{
		Type: xdr.ContractIdPreimageTypeContractIdPreimageFromAddress,
		FromAddress: &xdr.ContractIdPreimageFromAddress{
			Address: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &deployerContract,
			},
			Salt: xdr.Uint256([32]byte{0xAA}),
		},
	}

	expectedCreatedContractID := deriveContractIDString(t, networkPassphrase, preimage)
	expectedCreatedContractBytes := strkey.MustDecode(strkey.VersionByteContract, expectedCreatedContractID)
	var expectedCreatedContractHash xdr.Hash
	copy(expectedCreatedContractHash[:], expectedCreatedContractBytes)
	expectedCreatedContract := xdr.ContractId(expectedCreatedContractHash)

	if deployerContractID == expectedCreatedContractID {
		t.Fatal("test setup error: deployer and created contract IDs must differ")
	}

	deployerKey := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &deployerContract,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}
	createdContractKey := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &expectedCreatedContract,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}

	createContractOp := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeInvokeHostFunction,
			InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
				HostFunction: xdr.HostFunction{
					Type: xdr.HostFunctionTypeHostFunctionTypeCreateContract,
					CreateContract: &xdr.CreateContractArgs{
						ContractIdPreimage: preimage,
						Executable: xdr.ContractExecutable{
							Type:     xdr.ContractExecutableTypeContractExecutableWasm,
							WasmHash: &xdr.Hash{0xFF},
						},
					},
				},
			},
		},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{createContractOp},
			Ext: xdr.TransactionExt{
				V: 1,
				SorobanData: &xdr.SorobanTransactionData{
					Ext: xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{
						Footprint: xdr.LedgerFootprint{
							ReadWrite: []xdr.LedgerKey{deployerKey, createdContractKey},
						},
					},
					ResourceFee: 100,
				},
			},
		},
	}

	transaction := genericLedgerTransaction
	transaction.Envelope = xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1:   &envelope,
	}

	output, err := TransformOperation(createContractOp, 0, transaction, 0, makeLedgerCloseMeta(), networkPassphrase)
	if err != nil {
		t.Fatalf("TransformOperation failed: %v", err)
	}

	gotContractID, ok := output.OperationDetails["contract_id"].(string)
	if !ok || gotContractID == "" {
		t.Fatalf("contract_id missing from operation details: %#v", output.OperationDetails["contract_id"])
	}

	if gotContractID != deployerContractID {
		t.Fatalf("expected first footprint contract %q, got %q", deployerContractID, gotContractID)
	}

	if gotContractID == expectedCreatedContractID {
		t.Fatalf("expected wrong contract id, but got the created contract id %q", gotContractID)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `create_contract` and `create_contract_v2` should export the contract ID derived from the operation's `ContractIdPreimage` and the transaction network.
- **Actual**: the exporter returns the first `ContractData` contract ID present in the footprint, which can belong to a different contract involved in the transaction.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC derives the created contract ID from the canonical `HashIdPreimageContractId` hash and proves `TransformOperation()` exports a different, footprint-derived ID.
2. Realistic preconditions: YES — `FROM_ADDRESS` create-contract preimages can use a contract address, and a valid Soroban footprint can contain multiple `ContractData` entries for the transaction while the current code performs no filtering.
3. Bug vs by-design: BUG — the operation already carries the authoritative `ContractIdPreimage`; replacing it with the first transaction-footprint contract ID is lossy and undocumented.
4. Final severity: High — this corrupts structural contract metadata without directly changing monetary values.
5. In scope: YES — it is a concrete data-correctness bug in the export path under `internal/transform/`.
6. Test correctness: CORRECT — the final test removed the original PoC's loopholes by deriving the expected created contract ID, using a valid Soroban transaction extension (`V: 1`), and checking the transformed output rather than asserting on hard-coded fake data.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Derive `contract_id` for `create_contract` and `create_contract_v2` from `ContractIdPreimage` plus the network passphrase, matching Stellar's `HashIdPreimageContractId` construction. If a footprint fallback is kept, restrict it to the created contract's `SCV_LEDGER_KEY_CONTRACT_INSTANCE` key and use it only as validation, not as the primary source.
