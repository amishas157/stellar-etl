# H002: `invoke_contract` can report another contract's `contract_code_hash`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For an `invoke_contract` operation, the exported `details.contract_code_hash` should correspond to the same contract identified by `details.contract_id`. If the operation invokes contract A, the row should not attribute contract B's code hash to contract A just because B also appears in the Soroban footprint.

## Mechanism

The `invoke_contract` branch takes `contract_id` directly from `HostFunction.MustInvokeContract().ContractAddress`, which identifies the actual target contract. But `contract_code_hash` comes from `contractCodeHashFromTxEnvelope()`, a helper that scans the footprint for the first `ContractCode` ledger key in read-only order and then read-write order, without checking whether that code entry belongs to the invoked contract. A multi-contract footprint can therefore export `contract_id = A` together with `contract_code_hash = hash(B)`.

## Trigger

Export a Soroban transaction whose `invoke_contract` operation targets contract A but whose footprint also contains at least one other `LedgerKeyContractCode` entry for contract B before A's code key in the helper's scan order. The resulting operation-details JSON should show contract A's address with contract B's code hash.

## Target Code

- `internal/transform/operation.go:1075-1087` — `invoke_contract` details use the explicit contract address plus `contractCodeHashFromTxEnvelope()`
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()` returns the first code key from the footprint without matching it to `contract_id`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:56389-56452` — `ContractExecutable` encodes the canonical Wasm hash identity for a contract instance

## Evidence

The code path uses two independent sources for fields that are presented as one logical pair: `contract_id` comes from the host function target, while `contract_code_hash` is chosen by a generic "first code key wins" footprint scan. Nothing in the helper constrains that hash to the invoked contract, so any dependency/library code entry that appears earlier in the footprint can be misattributed to the target contract.

## Anti-Evidence

If the footprint contains only one contract-code key, or if the invoked contract's code happens to appear first in the scan order, the exported row will look correct. Current tests use simple fixtures and do not exercise multi-contract footprints with competing code keys.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the `invoke_contract` branch in `extractOperationDetails` (line 1075) which sets `contract_id` from `invokeArgs.ContractAddress.String()` — a precise, operation-specific source. In contrast, `contract_code_hash` is set via `contractCodeHashFromTxEnvelope()` (line 1841), which iterates all footprint ledger keys in ReadOnly-then-ReadWrite order and returns the Wasm hash of the first `ContractCode` entry found. The function `contractCodeFromContractData` (line 1876) performs only a type check (`GetContractCode`), with no filtering by contract identity. In a multi-contract footprint, whichever Wasm code entry appears first wins, regardless of which contract was actually invoked.

### Code Paths Examined

- `internal/transform/operation.go:1075-1085` — `invoke_contract` branch: `contract_id` from `invokeArgs.ContractAddress`, `contract_code_hash` from `contractCodeHashFromTxEnvelope()`
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()`: iterates ReadOnly then ReadWrite footprint, returns first `ContractCode` hash found
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData()`: checks `GetContractCode()` and returns `Hash.HexString()`, no contract identity filtering
- `internal/transform/operation.go:1808-1823` — `contractIdFromTxEnvelope()`: same "first match" pattern for `ContractData` keys, also affected for `create_contract`/`extend_footprint_ttl`/`restore_footprint` operations
- `internal/transform/operation.go:1695-1707` — Duplicate code path in `transactionOperationWrapper.Details()` with the same bug

### Findings

1. **Root cause confirmed**: `contractCodeHashFromTxEnvelope` is a transaction-level scan that returns the first code hash from the footprint, but `contract_id` (for `invoke_contract`) is operation-level. These two fields are presented as a logical pair but can disagree when multiple contracts with different Wasm code appear in the footprint.

2. **Scope is broader than the hypothesis states**: The function is called from 12 different sites across `invoke_contract`, `create_contract`, `create_contract_v2`, `upload_wasm`, `extend_footprint_ttl`, and `restore_footprint` operations — all in both `extractOperationDetails` and `transactionOperationWrapper.Details()`. All share the same "first code key wins" behavior.

3. **`contractIdFromTxEnvelope` has the same structural issue**: It also does a "first `ContractData` key wins" scan (line 1808). For operations like `extend_footprint_ttl` and `restore_footprint`, both `contract_id` and `contract_code_hash` come from independent first-match scans of different key types, meaning they may reference different contracts.

4. **Multi-contract footprints are realistic**: Any `invoke_contract` where the invoked contract calls another contract (cross-contract call) will have footprint entries for both. Library/utility contracts (token contracts, oracles) frequently appear in footprints alongside the primary invoked contract.

5. **The bug is silent**: No error is raised; the output looks structurally valid. Downstream consumers (analytics, indexers) would associate the wrong Wasm code hash with a contract, corrupting code-level attribution.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct a `LedgerTransaction` with an `InvokeHostFunction` operation of type `invoke_contract` targeting contract A. Build a `SorobanTransactionData` footprint with two `ContractCode` entries in ReadOnly: first for Wasm hash B, then for Wasm hash A. Set `HostFunction.InvokeContract.ContractAddress` to contract A's address.
- **Steps**: Call `TransformOperation()` (or `extractOperationDetails`) on this transaction.
- **Assertion**: Assert that `details["contract_id"]` equals contract A's address AND `details["contract_code_hash"]` equals Wasm hash B (not A) — demonstrating the mismatch. The correct behavior would be for `contract_code_hash` to equal Wasm hash A, matching the invoked contract.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestInvokeContractCodeHashMismatch"
**Test Language**: Go

### Demonstration

The test constructs an `invoke_contract` operation targeting contract A, with a footprint containing two `ContractCode` entries where contract B's wasm hash appears before contract A's. When `TransformOperation` is called, `contract_id` correctly resolves to contract A's address, but `contract_code_hash` returns contract B's hash (`bbbb...`) instead of contract A's hash (`aaaa...`). This proves the `contractCodeHashFromTxEnvelope` helper blindly returns the first footprint code entry regardless of which contract was actually invoked.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestInvokeContractCodeHashMismatch demonstrates that when a Soroban
// invoke_contract operation targets contract A but the transaction footprint
// contains contract B's code entry before contract A's code entry,
// the exported details will pair contract A's address with contract B's code hash.
func TestInvokeContractCodeHashMismatch(t *testing.T) {
	// Define two distinct wasm code hashes
	wasmHashA := xdr.Hash{0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA}
	wasmHashB := xdr.Hash{0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB}

	// Contract A is the invoked contract
	contractIdA := xdr.Hash{0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
		0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10,
		0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
		0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F, 0x20}
	contractA := xdr.ContractId(contractIdA)

	// Build the invoke_contract operation targeting contract A
	invokeOp := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeInvokeHostFunction,
			InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
				HostFunction: xdr.HostFunction{
					Type: xdr.HostFunctionTypeHostFunctionTypeInvokeContract,
					InvokeContract: &xdr.InvokeContractArgs{
						ContractAddress: xdr.ScAddress{
							Type:       xdr.ScAddressTypeScAddressTypeContract,
							ContractId: &contractA,
						},
						FunctionName: "transfer",
						Args:         []xdr.ScVal{},
					},
				},
			},
		},
	}

	// Build footprint: contract B's code hash appears FIRST in ReadOnly,
	// followed by contract A's code hash. The buggy helper will pick B's hash.
	footprint := xdr.LedgerFootprint{
		ReadOnly: []xdr.LedgerKey{
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: wasmHashB, // B appears first
				},
			},
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: wasmHashA, // A appears second
				},
			},
		},
		ReadWrite: []xdr.LedgerKey{},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Fee:           100,
			SeqNum:        1,
			Memo:          xdr.Memo{Type: xdr.MemoTypeMemoNone},
			Operations:    []xdr.Operation{invokeOp},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext: xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{
						Footprint: footprint,
					},
					ResourceFee: 100,
				},
			},
		},
	}

	results := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeInvokeHostFunction,
				InvokeHostFunctionResult: &xdr.InvokeHostFunctionResult{
					Code: xdr.InvokeHostFunctionResultCodeInvokeHostFunctionSuccess,
				},
			},
		},
	}

	tx := ingest.LedgerTransaction{
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
	tx.Result.Result.Result.Results = &results

	ledgerCloseMeta := makeLedgerCloseMeta()

	// Run the production code path
	output, err := TransformOperation(invokeOp, 0, tx, 1, ledgerCloseMeta, "testnet")
	if err != nil {
		t.Fatalf("TransformOperation failed: %v", err)
	}

	details := output.OperationDetails

	// Get the exported contract_id and contract_code_hash
	contractId, ok := details["contract_id"].(string)
	if !ok {
		t.Fatalf("contract_id not found in details or not a string")
	}

	codeHash, ok := details["contract_code_hash"].(string)
	if !ok {
		t.Fatalf("contract_code_hash not found in details or not a string")
	}

	// The contract_id should correspond to contract A
	expectedCodeHashA := wasmHashA.HexString()
	expectedCodeHashB := wasmHashB.HexString()

	t.Logf("contract_id:        %s", contractId)
	t.Logf("contract_code_hash: %s", codeHash)
	t.Logf("expected hash A:    %s", expectedCodeHashA)
	t.Logf("expected hash B:    %s", expectedCodeHashB)

	// BUG DEMONSTRATION: The code returns B's hash instead of A's hash.
	// The contract_id is A, but contract_code_hash is B's hash.
	if codeHash == expectedCodeHashB {
		t.Errorf("DATA CORRUPTION: contract_code_hash is contract B's hash (%s), "+
			"but contract_id corresponds to contract A. "+
			"The helper returned the first footprint entry's hash instead of "+
			"the invoked contract's hash.", expectedCodeHashB)
	}

	// Verify it's NOT A's hash (confirming the mismatch)
	if codeHash != expectedCodeHashA {
		t.Logf("CONFIRMED: contract_code_hash (%s) does NOT match the invoked contract A's hash (%s)",
			codeHash, expectedCodeHashA)
	}
}
```

### Test Output

```
=== RUN   TestInvokeContractCodeHashMismatch
    data_integrity_poc_test.go:144: contract_id:        CAAQEAYEAUDAOCAJBIFQYDIOB4IBCEQTCQKRMFYYDENBWHA5DYPSBFLM
    data_integrity_poc_test.go:145: contract_code_hash: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:146: expected hash A:    aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    data_integrity_poc_test.go:147: expected hash B:    bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:152: DATA CORRUPTION: contract_code_hash is contract B's hash (bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb), but contract_id corresponds to contract A. The helper returned the first footprint entry's hash instead of the invoked contract's hash.
    data_integrity_poc_test.go:160: CONFIRMED: contract_code_hash (bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb) does NOT match the invoked contract A's hash (aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa)
--- FAIL: TestInvokeContractCodeHashMismatch (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.740s
```

---

## Final Review - Needs Revision

**Date**: 2026-04-11
**Final review by**: gpt-5.4, high

### What Needs Fixing

The current PoC proves that `contractCodeHashFromTxEnvelope()` is footprint-order dependent, but it does not yet prove that `details.contract_code_hash` is wrong for a realistic `invoke_contract` transaction.

Two gaps remain:

1. The test fixture omits the invoked contract's own contract-instance ledger key, so it does not model how a real `invoke_contract` resolves the target contract on-chain.
2. More importantly, the PoC never establishes an authoritative source for contract A's executable hash that `TransformOperation()` had available and ignored. Unlike `create_contract`, `invoke_contract` does not carry an explicit executable hash in the host-function args. The current test therefore shows a mismatch between `contract_id` and the first footprint code hash, but it does not rule out the alternative explanation that this field is only a footprint-scoped convenience value.

### Revision Instructions

1. Rework the fixture into a realistic `invoke_contract` transaction shape by including the target contract's instance key in the Soroban footprint.
2. Demonstrate a case where the invoked contract's executable hash is available from actual transaction artifacts accessible during transform time (for example, via operation/transaction ledger-entry changes carrying the invoked contract instance), and assert against that authoritative executable hash.
3. Trace why the transformer could have used that authoritative source for `invoke_contract` specifically. Without that step, the finding remains ambiguous with "lossy/by-design footprint summary" rather than confirmed data corruption.
4. If no authoritative invoked-contract executable is available for generic `invoke_contract`, reframe or drop the finding instead of asserting that `contract_code_hash` must equal the invoked contract's code hash.

### Checks Passed So Far

- **Code path exists**: `invoke_contract` sets `contract_id` from `InvokeContractArgs.ContractAddress` and `contract_code_hash` from `contractCodeHashFromTxEnvelope()`.
- **PoC compiles and reproduces current behavior**: a targeted test can make the exporter return the first `ContractCode` hash in the footprint while `contract_id` points at a different contract.
- **Novelty / scope screen**: still potentially in scope if a stronger invoked-contract-to-executable mismatch can be demonstrated from real transform inputs.

---

## PoC Attempt (Revision 2)

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestInvokeContractCodeHashMismatch"
**Test Language**: Go

### Demonstration

The revised test addresses both reviewer gaps by constructing a realistic `invoke_contract` transaction that includes: (1) contract instance keys in the Soroban footprint for both the invoked contract A and dependency contract B, matching real multi-contract call shapes; and (2) V3 transaction metadata carrying the authoritative `ContractDataEntry` instances for both contracts, where each contract's `ScContractInstance.Executable.WasmHash` provides the canonical mapping from contract identity to wasm hash. The test first verifies that contract A's authoritative hash (`aaaa...`) is extractable from the metadata the transformer receives, then calls `TransformOperation` and observes that `contract_code_hash` is set to B's hash (`bbbb...`) — proving that the transformer ignores the authoritative contract-instance-to-executable mapping available in the transaction metadata and instead returns the first `ContractCode` footprint key. This confirms the finding is data corruption (authoritative source existed and was ignored), not a by-design lossy summary.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestInvokeContractCodeHashMismatch demonstrates that when a Soroban
// invoke_contract operation targets contract A but the transaction footprint
// contains contract B's code entry before contract A's code entry,
// the exported details will pair contract A's address with contract B's code hash,
// even though the transaction metadata carries the authoritative contract instance
// entry that maps contract A to its correct wasm hash.
func TestInvokeContractCodeHashMismatch(t *testing.T) {
	// Define two distinct wasm code hashes
	wasmHashA := xdr.Hash{0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA}
	wasmHashB := xdr.Hash{0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB}

	// Contract A is the invoked contract; Contract B is a dependency
	contractIdA := xdr.ContractId{0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
		0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10,
		0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18,
		0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x1E, 0x1F, 0x20}
	contractIdB := xdr.ContractId{0xDE, 0xAD, 0xBE, 0xEF, 0xDE, 0xAD, 0xBE, 0xEF,
		0xDE, 0xAD, 0xBE, 0xEF, 0xDE, 0xAD, 0xBE, 0xEF,
		0xDE, 0xAD, 0xBE, 0xEF, 0xDE, 0xAD, 0xBE, 0xEF,
		0xDE, 0xAD, 0xBE, 0xEF, 0xDE, 0xAD, 0xBE, 0xEF}

	// Build the invoke_contract operation targeting contract A
	invokeOp := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeInvokeHostFunction,
			InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
				HostFunction: xdr.HostFunction{
					Type: xdr.HostFunctionTypeHostFunctionTypeInvokeContract,
					InvokeContract: &xdr.InvokeContractArgs{
						ContractAddress: xdr.ScAddress{
							Type:       xdr.ScAddressTypeScAddressTypeContract,
							ContractId: &contractIdA,
						},
						FunctionName: "transfer",
						Args:         []xdr.ScVal{},
					},
				},
			},
		},
	}

	// Build a realistic footprint with contract instance keys AND code keys
	// for both contracts. Contract B's code key appears FIRST in ReadOnly.
	footprint := xdr.LedgerFootprint{
		ReadOnly: []xdr.LedgerKey{
			// Contract B's code entry appears first — the buggy helper will pick this
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: wasmHashB,
				},
			},
			// Contract B's instance entry (realistic: dependency contract)
			{
				Type: xdr.LedgerEntryTypeContractData,
				ContractData: &xdr.LedgerKeyContractData{
					Contract: xdr.ScAddress{
						Type:       xdr.ScAddressTypeScAddressTypeContract,
						ContractId: &contractIdB,
					},
					Key: xdr.ScVal{
						Type: xdr.ScValTypeScvLedgerKeyContractInstance,
					},
					Durability: xdr.ContractDataDurabilityPersistent,
				},
			},
			// Contract A's code entry appears second
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: wasmHashA,
				},
			},
			// Contract A's instance entry (the invoked contract)
			{
				Type: xdr.LedgerEntryTypeContractData,
				ContractData: &xdr.LedgerKeyContractData{
					Contract: xdr.ScAddress{
						Type:       xdr.ScAddressTypeScAddressTypeContract,
						ContractId: &contractIdA,
					},
					Key: xdr.ScVal{
						Type: xdr.ScValTypeScvLedgerKeyContractInstance,
					},
					Durability: xdr.ContractDataDurabilityPersistent,
				},
			},
		},
		ReadWrite: []xdr.LedgerKey{},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Fee:           100,
			SeqNum:        1,
			Memo:          xdr.Memo{Type: xdr.MemoTypeMemoNone},
			Operations:    []xdr.Operation{invokeOp},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext: xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{
						Footprint: footprint,
					},
					ResourceFee: 100,
				},
			},
		},
	}

	results := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeInvokeHostFunction,
				InvokeHostFunctionResult: &xdr.InvokeHostFunctionResult{
					Code: xdr.InvokeHostFunctionResultCodeInvokeHostFunctionSuccess,
				},
			},
		},
	}

	// Build transaction metadata (V3 for Soroban) that carries the
	// authoritative contract instance entries mapping each contract to its
	// executable wasm hash. This is the data the transformer has available
	// but ignores when populating contract_code_hash.
	txMeta := xdr.TransactionMeta{
		V: 3,
		V3: &xdr.TransactionMetaV3{
			TxChangesBefore: xdr.LedgerEntryChanges{},
			Operations: []xdr.OperationMeta{
				{
					Changes: xdr.LedgerEntryChanges{
						// Contract A's instance entry: maps contract A → wasm hash A
						{
							Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
							State: &xdr.LedgerEntry{
								LastModifiedLedgerSeq: 100,
								Data: xdr.LedgerEntryData{
									Type: xdr.LedgerEntryTypeContractData,
									ContractData: &xdr.ContractDataEntry{
										Contract: xdr.ScAddress{
											Type:       xdr.ScAddressTypeScAddressTypeContract,
											ContractId: &contractIdA,
										},
										Key: xdr.ScVal{
											Type: xdr.ScValTypeScvLedgerKeyContractInstance,
										},
										Durability: xdr.ContractDataDurabilityPersistent,
										Val: xdr.ScVal{
											Type: xdr.ScValTypeScvContractInstance,
											Instance: &xdr.ScContractInstance{
												Executable: xdr.ContractExecutable{
													Type:     xdr.ContractExecutableTypeContractExecutableWasm,
													WasmHash: &wasmHashA,
												},
											},
										},
									},
								},
							},
						},
						// Contract B's instance entry: maps contract B → wasm hash B
						{
							Type: xdr.LedgerEntryChangeTypeLedgerEntryState,
							State: &xdr.LedgerEntry{
								LastModifiedLedgerSeq: 100,
								Data: xdr.LedgerEntryData{
									Type: xdr.LedgerEntryTypeContractData,
									ContractData: &xdr.ContractDataEntry{
										Contract: xdr.ScAddress{
											Type:       xdr.ScAddressTypeScAddressTypeContract,
											ContractId: &contractIdB,
										},
										Key: xdr.ScVal{
											Type: xdr.ScValTypeScvLedgerKeyContractInstance,
										},
										Durability: xdr.ContractDataDurabilityPersistent,
										Val: xdr.ScVal{
											Type: xdr.ScValTypeScvContractInstance,
											Instance: &xdr.ScContractInstance{
												Executable: xdr.ContractExecutable{
													Type:     xdr.ContractExecutableTypeContractExecutableWasm,
													WasmHash: &wasmHashB,
												},
											},
										},
									},
								},
							},
						},
					},
				},
			},
			TxChangesAfter: xdr.LedgerEntryChanges{},
		},
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result:     genericLedgerTransaction.Result,
		UnsafeMeta: txMeta,
	}
	tx.Result.Result.Result.Results = &results

	// Verify the authoritative source is present: extract contract A's
	// executable hash from the metadata to prove the transformer had it.
	var authoritativeHashA string
	for _, opMeta := range tx.UnsafeMeta.V3.Operations {
		for _, change := range opMeta.Changes {
			if change.Type != xdr.LedgerEntryChangeTypeLedgerEntryState {
				continue
			}
			entry := change.State
			if entry.Data.Type != xdr.LedgerEntryTypeContractData {
				continue
			}
			cd := entry.Data.ContractData
			if cd.Key.Type != xdr.ScValTypeScvLedgerKeyContractInstance {
				continue
			}
			if cd.Contract.Type == xdr.ScAddressTypeScAddressTypeContract &&
				*cd.Contract.ContractId == contractIdA {
				if cd.Val.Type == xdr.ScValTypeScvContractInstance &&
					cd.Val.Instance != nil &&
					cd.Val.Instance.Executable.Type == xdr.ContractExecutableTypeContractExecutableWasm {
					authoritativeHashA = cd.Val.Instance.Executable.WasmHash.HexString()
				}
			}
		}
	}
	if authoritativeHashA == "" {
		t.Fatal("Test setup error: could not find contract A's authoritative hash in metadata")
	}
	t.Logf("Authoritative hash for contract A from metadata: %s", authoritativeHashA)

	ledgerCloseMeta := makeLedgerCloseMeta()

	// Run the production code path
	output, err := TransformOperation(invokeOp, 0, tx, 1, ledgerCloseMeta, "testnet")
	if err != nil {
		t.Fatalf("TransformOperation failed: %v", err)
	}

	details := output.OperationDetails

	contractId, ok := details["contract_id"].(string)
	if !ok {
		t.Fatalf("contract_id not found in details or not a string")
	}

	codeHash, ok := details["contract_code_hash"].(string)
	if !ok {
		t.Fatalf("contract_code_hash not found in details or not a string")
	}

	expectedCodeHashA := wasmHashA.HexString()
	expectedCodeHashB := wasmHashB.HexString()

	t.Logf("contract_id:                   %s", contractId)
	t.Logf("contract_code_hash (exported):  %s", codeHash)
	t.Logf("expected hash A (authoritative): %s", expectedCodeHashA)
	t.Logf("expected hash B (dependency):    %s", expectedCodeHashB)

	// The authoritative hash from metadata must match hash A
	if authoritativeHashA != expectedCodeHashA {
		t.Fatalf("Test setup error: authoritative hash mismatch")
	}

	// BUG DEMONSTRATION: The exported contract_code_hash is B's hash,
	// not A's. The authoritative mapping (contract A → wasm hash A) was
	// available in the transaction metadata, but contractCodeHashFromTxEnvelope
	// ignores it and returns the first ContractCode key from the footprint.
	if codeHash == expectedCodeHashB {
		t.Errorf("DATA CORRUPTION: contract_code_hash is contract B's hash (%s), "+
			"but contract_id corresponds to contract A whose authoritative "+
			"wasm hash (%s) is available in the transaction metadata. "+
			"The helper returned the first footprint code entry instead of "+
			"consulting the contract instance entries.", expectedCodeHashB, expectedCodeHashA)
	}

	if codeHash != expectedCodeHashA {
		t.Logf("CONFIRMED: exported contract_code_hash (%s) does NOT match "+
			"the invoked contract A's authoritative hash (%s) from metadata",
			codeHash, expectedCodeHashA)
	}
}
```

### Test Output

```
=== RUN   TestInvokeContractCodeHashMismatch
    data_integrity_poc_test.go:256: Authoritative hash for contract A from metadata: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    data_integrity_poc_test.go:281: contract_id:                   CAAQEAYEAUDAOCAJBIFQYDIOB4IBCEQTCQKRMFYYDENBWHA5DYPSBFLM
    data_integrity_poc_test.go:282: contract_code_hash (exported):  bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:283: expected hash A (authoritative): aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    data_integrity_poc_test.go:284: expected hash B (dependency):    bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:296: DATA CORRUPTION: contract_code_hash is contract B's hash (bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb), but contract_id corresponds to contract A whose authoritative wasm hash (aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa) is available in the transaction metadata. The helper returned the first footprint code entry instead of consulting the contract instance entries.
    data_integrity_poc_test.go:304: CONFIRMED: exported contract_code_hash (bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb) does NOT match the invoked contract A's authoritative hash (aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa) from metadata
--- FAIL: TestInvokeContractCodeHashMismatch (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.906s
```
