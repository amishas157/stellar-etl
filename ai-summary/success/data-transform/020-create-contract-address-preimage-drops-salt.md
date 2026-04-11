# 020: Create-contract address preimage drops salt

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports `create_contract` and `create_contract_v2` details by passing `ContractIdPreimage` through `switchContractIdPreimageType()`. For `CONTRACT_ID_PREIMAGE_FROM_ADDRESS`, that helper emits only `from=address` and the deployer address, silently dropping the required 256-bit `salt` even though it is part of the on-chain preimage.

This produces plausible but incomplete `history_operations.details` rows. Downstream consumers can see that the contract was created from an address preimage, but they cannot recover the full preimage that actually distinguished one deployment from another.

## Root Cause

Both create-contract branches merge the map returned by `switchContractIdPreimageType()` into `OperationDetails`. In the `FROM_ADDRESS` case, the helper reads `fromAddress.Address` and never exports `fromAddress.Salt`, so the transformer loses part of the XDR payload before serializing the row.

## Reproduction

Create a successful Soroban `create_contract` operation whose `ContractIdPreimage` is `FROM_ADDRESS` and whose `Salt` is non-zero. Run the operation through `TransformOperation()` and inspect the exported `OperationDetails`.

The output includes `from` and `address` but no `salt`, even though the input XDR preimage contains both fields and Horizon's canonical operation details preserve the salt for the same operation shape.

## Affected Code

- `internal/transform/operation.go:1099-1113` — `extractOperationDetails()` copies the lossy preimage map into `create_contract` details
- `internal/transform/operation.go:1121-1140` — `extractOperationDetails()` does the same for `create_contract_v2`
- `internal/transform/operation.go:1721-1735` — duplicate `transactionOperationWrapper.Details()` path repeats the same omission for `create_contract`
- `internal/transform/operation.go:1743-1762` — duplicate `transactionOperationWrapper.Details()` path repeats it for `create_contract_v2`
- `internal/transform/operation.go:2275-2294` — `switchContractIdPreimageType()` drops `ContractIdPreimageFromAddress.Salt`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestCreateContractFromAddressDropsSalt`
- **Test language**: go
- **How to run**: Create the target test file with the test body below, then run `go build ./...` and `go test ./internal/transform/... -run TestCreateContractFromAddressDropsSalt -v`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestCreateContractFromAddressDropsSalt(t *testing.T) {
	salt := xdr.Uint256{
		0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
		0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f, 0x10,
		0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
		0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f, 0x20,
	}
	expectedSaltStr := salt.String()

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
									ContractId: &xdr.ContractId{},
								},
								Salt: salt,
							},
						},
						Executable: xdr.ContractExecutable{},
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
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext: xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{
						Footprint: xdr.LedgerFootprint{ReadOnly: []xdr.LedgerKey{}, ReadWrite: []xdr.LedgerKey{}},
					},
					ResourceFee: 100,
				},
			},
		},
	}

	txn := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{Type: xdr.EnvelopeTypeEnvelopeTypeTx, V1: &envelope},
		Result:     genericLedgerTransaction.Result,
		UnsafeMeta: genericLedgerTransaction.UnsafeMeta,
	}

	ledgerCloseMeta := makeLedgerCloseMeta()
	output, err := TransformOperation(createContractOp, 0, txn, 0, ledgerCloseMeta, "")
	if err != nil {
		t.Fatalf("TransformOperation returned unexpected error: %v", err)
	}

	saltVal, hasSalt := output.OperationDetails["salt"]
	if !hasSalt {
		t.Fatalf("operation details missing salt; expected %s; got keys %v", expectedSaltStr, keys(output.OperationDetails))
	}
	if saltVal != expectedSaltStr {
		t.Fatalf("salt mismatch: got %v want %s", saltVal, expectedSaltStr)
	}
}

func keys(m map[string]interface{}) []string {
	ks := make([]string, 0, len(m))
	for k := range m {
		ks = append(ks, k)
	}
	return ks
}
```

## Expected vs Actual Behavior

- **Expected**: address-based `create_contract` and `create_contract_v2` exports should preserve both the deployer address and the preimage salt from `ContractIdPreimageFromAddress`.
- **Actual**: the exporter preserves only `from=address` and `address`, dropping the salt from every address-based contract-creation row.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC constructs a real address-based create-contract operation, runs the production `TransformOperation()` path, and proves the `salt` field is absent from `OperationDetails`.
2. Realistic preconditions: YES — `ContractIdPreimageFromAddress` is a normal Soroban contract-creation mode, and `Salt` is a mandatory member of that XDR arm.
3. Bug vs by-design: BUG — the omission loses a first-class preimage input from exported operation details, and Horizon's canonical operation details include `salt` for the same `FROM_ADDRESS` cases.
4. Final severity: High — this corrupts structural contract-creation metadata without directly changing monetary values.
5. In scope: YES — it is a concrete export-path data correctness issue in `internal/transform/`.
6. Test correctness: CORRECT — the assertion checks the transformed output, not test setup, and fails only because the production helper omits the field.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

In `switchContractIdPreimageType()`, add `details["salt"] = fromAddress.Salt.String()` in the `ContractIdPreimageTypeContractIdPreimageFromAddress` case. Mirror that behavior anywhere the duplicate operation-detail formatter is maintained so both paths stay consistent.
