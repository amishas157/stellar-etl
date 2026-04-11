# 015: Create-contract ignores executable Wasm hash

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`extractOperationDetails()` and the duplicate `transactionOperationWrapper.Details()` path populate `contract_code_hash` for `create_contract` and `create_contract_v2` by scanning the transaction footprint and returning the first `ContractCode` ledger key they find. The host-function args already carry the authoritative per-operation executable hash in `args.Executable.WasmHash`, so any create-contract transaction whose footprint contains another `ContractCode` entry ahead of the target Wasm exports the wrong hash.

The original hypothesis overstated the trigger by describing two create operations in one transaction, which Soroban does not allow. The core finding is still real: a single create-contract operation can legitimately involve multiple `ContractCode` keys, and the exporter silently binds the row to whichever one sorts first rather than the Wasm hash the operation actually requested.

## Root Cause

Both create-contract branches ignore `CreateContractArgs.Executable` / `CreateContractArgsV2.Executable` entirely and call `contractCodeHashFromTxEnvelope()` instead. That helper walks `ReadOnly` first and `ReadWrite` second and returns the first `LedgerKeyContractCode.Hash` it encounters, so it discards the operation-scoped executable hash in favor of a transaction-scoped, order-dependent surrogate.

## Reproduction

During normal Soroban operation, create-contract calls can be made on behalf of a contract address, and Soroban supports contract-driven deployments (`with_address(...).deploy_v2(...)`). In that shape, the transaction can legitimately carry more than one `ContractCode` ledger key in its footprint while still having exactly one create-contract operation.

When the deployer contract's code hash sorts ahead of the target Wasm hash, the current transformer exports the deployer hash as `contract_code_hash`. Downstream consumers see a plausible 64-byte hash string, but it identifies the wrong contract code for the create operation.

## Affected Code

- `internal/transform/operation.go:1099-1127` — `extractOperationDetails()` assigns `contract_code_hash` for `create_contract` and `create_contract_v2` from the transaction footprint helper instead of `args.Executable`
- `internal/transform/operation.go:1721-1749` — `transactionOperationWrapper.Details()` duplicates the same lossy mapping
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()` returns the first footprint `ContractCode` key, not the operation's executable hash

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestCreateContractIgnoresExecutableWasmHash`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestCreateContractIgnoresExecutableWasmHash -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestCreateContractIgnoresExecutableWasmHash(t *testing.T) {
	targetWasmHash := xdr.Hash{
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
	}
	deployerWasmHash := xdr.Hash{
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
	}
	salt := xdr.Uint256([32]byte{0x01})
	deployerContractID := xdr.ContractId{
		0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC,
		0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC,
		0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC,
		0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC, 0xCC,
	}

	createContractOp := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeInvokeHostFunction,
			InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
				HostFunction: xdr.HostFunction{
					Type: xdr.HostFunctionTypeHostFunctionTypeCreateContract,
					CreateContract: &xdr.CreateContractArgs{
						ContractIdPreimage: xdr.ContractIdPreimage{
							Type: xdr.ContractIdPreimageTypeContractIdPreimageFromAddress,
							FromAddress: &xdr.ContractIdPreimageFromAddress{
								Address: xdr.ScAddress{
									Type:       xdr.ScAddressTypeScAddressTypeContract,
									ContractId: &deployerContractID,
								},
								Salt: salt,
							},
						},
						Executable: xdr.ContractExecutable{
							Type:     xdr.ContractExecutableTypeContractExecutableWasm,
							WasmHash: &targetWasmHash,
						},
					},
				},
			},
		},
	}

	envelope := xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1: &xdr.TransactionV1Envelope{
			Tx: xdr.Transaction{
				SourceAccount: genericSourceAccount,
				Operations:    []xdr.Operation{createContractOp},
				Ext: xdr.TransactionExt{
					V: 0,
					SorobanData: &xdr.SorobanTransactionData{
						Resources: xdr.SorobanResources{
							Footprint: xdr.LedgerFootprint{
								ReadOnly: []xdr.LedgerKey{
									{
										Type: xdr.LedgerEntryTypeContractCode,
										ContractCode: &xdr.LedgerKeyContractCode{Hash: deployerWasmHash},
									},
									{
										Type: xdr.LedgerEntryTypeContractCode,
										ContractCode: &xdr.LedgerKeyContractCode{Hash: targetWasmHash},
									},
								},
							},
						},
					},
				},
			},
		},
	}

	transaction := ingest.LedgerTransaction{
		Index:      1,
		Envelope:   envelope,
		Result:     genericLedgerTransaction.Result,
		UnsafeMeta: genericLedgerTransaction.UnsafeMeta,
	}

	details, err := extractOperationDetails(createContractOp, transaction, 0, "testnet")
	if err != nil {
		t.Fatalf("extractOperationDetails failed: %v", err)
	}

	actualHash, ok := details["contract_code_hash"].(string)
	if !ok {
		t.Fatalf("contract_code_hash missing from details: %#v", details["contract_code_hash"])
	}

	expectedCorrectHash := createContractOp.Body.MustInvokeHostFunctionOp().HostFunction.MustCreateContract().Executable.MustWasmHash().HexString()
	expectedBuggyHash := deployerWasmHash.HexString()

	if expectedCorrectHash != targetWasmHash.HexString() {
		t.Fatalf("unexpected executable wasm hash: got %s want %s", expectedCorrectHash, targetWasmHash.HexString())
	}
	if actualHash != expectedBuggyHash {
		t.Fatalf("expected first footprint ContractCode hash %s, got %s", expectedBuggyHash, actualHash)
	}
	if actualHash == expectedCorrectHash {
		t.Fatalf("expected exported hash to differ from the operation executable hash %s", expectedCorrectHash)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `create_contract` and `create_contract_v2` should export the Wasm hash carried in `args.Executable`, which is the code hash the operation is creating the contract from.
- **Actual**: The exporter returns the first `ContractCode` hash present in the transaction footprint, which can belong to a different contract involved in the same Soroban transaction.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `extractOperationDetails()` directly on a real `create_contract` operation and shows that the exported hash comes from `contractCodeHashFromTxEnvelope()`, not `args.Executable.WasmHash`.
2. Realistic preconditions: YES — the original multi-operation trigger was incorrect, but Soroban does support contract-address deployers and contract-driven deployments, which makes multi-`ContractCode` footprints for a single create operation realistic.
3. Bug vs by-design: BUG — the operation already includes an authoritative per-operation executable hash, so replacing it with the first transaction-footprint code hash is unnecessary, lossy, and can diverge from the actual deployed Wasm.
4. Final severity: High — this corrupts structural contract metadata (`contract_code_hash`) without directly changing monetary fields.
5. In scope: YES — this is a concrete export-path data corruption issue in `internal/transform/`.
6. Test correctness: CORRECT — the test asserts both the authoritative executable hash and the exported hash, and it fails unless the transformer returns a different footprint-derived value.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

For `HostFunctionTypeCreateContract` and `HostFunctionTypeCreateContractV2`, populate `contract_code_hash` from `args.Executable` when the executable arm is `CONTRACT_EXECUTABLE_WASM`, falling back to the footprint helper only for operations that do not carry an explicit Wasm hash.
