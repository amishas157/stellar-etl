# H004: `create_contract` exports the first footprint contract ID instead of the newly created contract

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: operation detail corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`create_contract` and `create_contract_v2` should export the contract ID of the contract being created. That value is deterministically tied to the operation's `ContractIdPreimage` and to the newly created `ContractInstance` key that must appear in the Soroban footprint, so unrelated pre-existing contract-data keys in the same footprint should not change the exported `contract_id`.

## Mechanism

Both create-contract branches populate `details["contract_id"]` by calling `contractIdFromTxEnvelope()`, which simply returns the first `ContractData` ledger key it sees in `ReadWrite` and then `ReadOnly`. If the transaction footprint also includes storage for a factory/deployer/SAC contract ahead of the new `ContractInstance`, the ETL exports that existing contract's ID instead of the newly created contract's ID.

## Trigger

Export a successful `create_contract` or `create_contract_v2` transaction whose Soroban footprint contains at least one pre-existing `ContractData` key before the new contract-instance key. The operation row will contain a well-formed `contract_id`, but it will point at the pre-existing contract rather than the contract created by this operation.

## Target Code

- `internal/transform/operation.go:1099-1127` — both create-contract branches fill `contract_id` from the footprint scanner
- `internal/transform/operation.go:1808-1824` — `contractIdFromTxEnvelope()` returns the first matching `ContractData` key in footprint order
- `internal/transform/operation.go:1108-1113` — `create_contract` already has the authoritative `ContractIdPreimage`, but that data is not used to derive or validate the exported contract ID
- `internal/transform/operation.go:1135-1140` — same omission for `create_contract_v2`

## Evidence

Unlike the already-published Wasm-hash bug, there is no alternate source for `contract_id` in the current implementation: the transformer never derives it from the preimage and never verifies that the scanned footprint key is the newly created instance. The exported value is therefore completely dependent on footprint order whenever more than one `ContractData` key is present.

## Anti-Evidence

If the new contract instance is the only `ContractData` key, or happens to be first in footprint order, the row is correct. The earlier rejected "empty contract_id" investigations do not rule out this stronger wrong-first-match case.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `extractOperationDetails()` for both `HostFunctionTypeCreateContract` (line 1099) and `HostFunctionTypeCreateContractV2` (line 1119), confirming both populate `details["contract_id"]` via `contractIdFromTxEnvelope()`. That helper (lines 1808-1824) iterates ReadWrite then ReadOnly footprint keys and returns the first `ContractData` match via `contractIdFromContractData()` (lines 1826-1839), which extracts the contract ID from ANY `ContractData` entry — not just `ContractInstance` keys. The duplicate `transactionOperationWrapper.Details()` path at lines 1727 and 1748 has the identical issue.

### Code Paths Examined

- `internal/transform/operation.go:1099-1113` — `create_contract` branch: calls `contractIdFromTxEnvelope(transactionEnvelope)` at line 1105, then calls `switchContractIdPreimageType(args.ContractIdPreimage)` at line 1108 which exports `from`/`address`/`asset` but NOT the derived contract ID
- `internal/transform/operation.go:1119-1140` — `create_contract_v2` branch: identical pattern, `contractIdFromTxEnvelope()` at line 1126
- `internal/transform/operation.go:1808-1824` — `contractIdFromTxEnvelope()`: iterates ReadWrite first, then ReadOnly, returns first `ContractData` match
- `internal/transform/operation.go:1826-1839` — `contractIdFromContractData()`: matches ANY `ContractData` key with a `ContractId` in its `Contract` field — no filtering for `ContractInstance` key type
- `internal/transform/operation.go:1721-1762` — duplicate `transactionOperationWrapper.Details()` path: identical first-match footprint scanning at lines 1727 and 1748
- `internal/transform/operation.go:2275-2295` — `switchContractIdPreimageType()`: extracts preimage metadata (`from`, `address`, `asset`) but does NOT derive the actual contract ID from the preimage
- `internal/transform/operation.go:1070-1084` — `invoke_contract` case: correctly uses `invokeArgs.ContractAddress.String()` for `contract_id` from the operation args, proving the developers know how to use per-operation sources when available

### Findings

1. **Authoritative per-operation source exists but is bypassed.** The `CreateContractArgs.ContractIdPreimage` (and `CreateContractArgsV2.ContractIdPreimage`) deterministically identify the contract being created. The Stellar protocol defines the contract ID as `SHA256(network_id + preimage_type + preimage_data)`. The `network` parameter is already available to `extractOperationDetails()` (line 584). Neither the primary nor the duplicate code path uses this authoritative source.

2. **First-match scanner is provably wrong for multi-contract footprints.** In factory/deployer patterns, the deployer contract modifies its own storage during the create operation. Those `ContractData` entries appear in the ReadWrite footprint. If any deployer `ContractData` entry is listed before the new contract's `ContractInstance` entry, `contractIdFromTxEnvelope()` returns the deployer's contract ID instead.

3. **Consistent with success/015 precedent.** Success/015 confirmed the identical class of bug for `contract_code_hash`: the footprint scanner returns the first match instead of using the authoritative per-operation source (`args.Executable.WasmHash`). H004 is the sibling bug for `contract_id` with a different authoritative source (`args.ContractIdPreimage`).

4. **Distinct from fail/045-046.** Those entries were rejected because they claimed the scanner returns EMPTY results (missing contract in footprint). H004 claims the scanner returns WRONG results (another contract's ID comes first). The exhaustive footprint guarantee from 045/046 ensures a match always exists, but does not guarantee it's the FIRST match.

5. **Distinct from fail/066.** That entry covered TTL operations, which have no authoritative per-operation contract source. For `create_contract`, the `ContractIdPreimage` IS authoritative.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Construct a `create_contract` operation with `ContractIdPreimage` of type `FROM_ADDRESS` (deployer + salt). Build a `TransactionV1Envelope` whose `ReadWrite` footprint contains a deployer's `ContractData` entry (arbitrary key, deployer contract ID) BEFORE the new contract's `ContractInstance` entry (new contract ID). Use distinct contract IDs for the deployer and the new contract.
- **Steps**: Call `extractOperationDetails()` with the constructed operation and transaction.
- **Assertion**: Assert that `details["contract_id"]` equals the deployer's contract ID (demonstrating the bug), NOT the new contract's ID. Optionally derive the expected contract ID from the preimage using `SHA256(network_passphrase + preimage_type_byte + preimage_data)` and show it differs from the exported value.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestCreateContractUsesFirstFootprintContract"
**Test Language**: Go

### Demonstration

The test constructs a `create_contract` operation with a footprint containing two `ContractData` entries: a deployer's entry (contract ID `0x01...`) listed first, followed by the new contract's entry (contract ID `0x02...`). When `extractOperationDetails()` is called, it returns the deployer's contract ID as `details["contract_id"]` because `contractIdFromTxEnvelope()` returns the first `ContractData` match in footprint order, ignoring the authoritative `ContractIdPreimage` in the operation args.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestCreateContractUsesFirstFootprintContract demonstrates that
// contractIdFromTxEnvelope returns the first ContractData match in the
// footprint rather than the contract being created. When a deployer contract's
// storage entry appears before the new ContractInstance entry, the exported
// contract_id is the deployer's — not the newly created contract's.
func TestCreateContractUsesFirstFootprintContract(t *testing.T) {
	// Two distinct contract IDs: a deployer and the new contract
	deployerHash := xdr.Hash{0x01}
	newContractHash := xdr.Hash{0x02}
	deployerCID := xdr.ContractId(deployerHash)
	newCID := xdr.ContractId(newContractHash)

	deployerContractID := strkey.MustEncode(strkey.VersionByteContract, deployerHash[:])
	newContractID := strkey.MustEncode(strkey.VersionByteContract, newContractHash[:])

	if deployerContractID == newContractID {
		t.Fatal("test setup error: deployer and new contract IDs must differ")
	}

	// Build footprint: deployer's ContractData BEFORE the new contract's ContractInstance
	deployerKey := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &deployerCID,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}
	newContractKey := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &newCID,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}

	// Build the create_contract operation
	salt := xdr.Uint256([32]byte{0xAA})
	sourceAccount := genericSourceAccount
	createContractOp := xdr.Operation{
		SourceAccount: &sourceAccount,
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
									ContractId: &deployerCID,
								},
								Salt: salt,
							},
						},
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
			SourceAccount: sourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{createContractOp},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Resources: xdr.SorobanResources{
						Footprint: xdr.LedgerFootprint{
							ReadWrite: []xdr.LedgerKey{deployerKey, newContractKey},
							ReadOnly:  []xdr.LedgerKey{},
						},
					},
					ResourceFee: 100,
				},
			},
		},
	}

	transaction := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: genericLedgerTransaction.Result,
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: genericTxMeta,
		},
	}

	// Run the production code path
	details, err := extractOperationDetails(createContractOp, transaction, 0, "Test SDF Network ; September 2015")
	if err != nil {
		t.Fatalf("extractOperationDetails failed: %v", err)
	}

	contractID, ok := details["contract_id"].(string)
	if !ok || contractID == "" {
		t.Fatal("contract_id not found in operation details")
	}

	// The bug: contract_id matches the deployer (first footprint entry),
	// not the newly created contract.
	if contractID == deployerContractID {
		t.Errorf("BUG CONFIRMED: contract_id is the deployer's (%s), not the new contract's (%s).\n"+
			"contractIdFromTxEnvelope() returns the first ContractData match in footprint order.",
			deployerContractID, newContractID)
	}

	// If the code were correct, contract_id would equal newContractID.
	if contractID == newContractID {
		t.Log("contract_id correctly matches the new contract — hypothesis NOT demonstrated")
	}
}
```

### Test Output

```
=== RUN   TestCreateContractUsesFirstFootprintContract
    data_integrity_poc_test.go:132: BUG CONFIRMED: contract_id is the deployer's (CAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABDQF), not the new contract's (CABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAARHO).
        contractIdFromTxEnvelope() returns the first ContractData match in footprint order.
--- FAIL: TestCreateContractUsesFirstFootprintContract (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.818s
```
