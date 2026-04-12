# H003: `extend_footprint_ttl` and `restore_footprint` export arbitrary first-match contract metadata

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `extend_footprint_ttl` and `restore_footprint`, exported `details.contract_id` and `details.contract_code_hash` should describe the actual target of the operation, or remain empty when the footprint spans multiple contracts/code entries and no single authoritative value exists. The transform should not fabricate a singular contract identifier by picking whichever matching footprint entry happens to appear first.

## Mechanism

Both operation cases populate `ledger_key_hash` with the full footprint set, then separately fill `contract_id` and `contract_code_hash` by scanning the entire Soroban footprint and returning the first matching `ContractData` or `ContractCode` key. These TTL operations apply to a footprint set, not a uniquely identified contract, so a legitimate footprint containing keys for multiple contracts or unrelated contract-code entries will export a plausible but wrong singular `contract_id` / `contract_code_hash` that depends on footprint ordering rather than the actual restored/extended target set.

## Trigger

Construct a Soroban `extend_footprint_ttl` or `restore_footprint` transaction whose footprint contains keys for contract A and contract B, or a contract-data key plus an unrelated contract-code key that sorts earlier in the helper scan order. Export the operation: `details.ledger_key_hash` will show the full set, but `details.contract_id` and/or `details.contract_code_hash` will name only the first matching footprint entry.

## Target Code

- `internal/transform/operation.go:1144-1159` — `extend_footprint_ttl` and `restore_footprint` assign singular contract metadata from footprint-wide helper scans.
- `internal/transform/operation.go:1808-1856` — `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` return the first match found in the footprint.

## Evidence

The helpers do not inspect which ledger key the TTL operation is semantically about; they just scan `ReadWrite` / `ReadOnly` arrays and return on the first match. The same operation details already export `ledger_key_hash` as a list, which is strong evidence that the underlying target is a set of keys rather than one canonical contract identifier.

## Anti-Evidence

Single-contract footprints will look correct, and some downstream consumers may treat these singular fields as heuristics. But the current export still presents them as authoritative scalar metadata even when the footprint legitimately spans multiple contracts or code entries.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Success 024 covers `create_contract`/`create_contract_v2` using the same helper but does NOT cover `extend_footprint_ttl` or `restore_footprint`. Fail 047 addresses a different `restore_footprint` issue (SAC balance contract attribution).

### Trace Summary

Traced from `extractOperationDetails()` through the `extend_footprint_ttl` (line 1144) and `restore_footprint` (line 1153) cases. Both call `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()`, which iterate the Soroban footprint's `ReadWrite` then `ReadOnly` arrays and return the first `ContractData` or `ContractCode` match respectively. In contrast, `ledgerKeyHashFromTxEnvelope()` (line 1859) correctly returns ALL ledger key hashes as a `[]string` slice, confirming the underlying data is set-valued. The singular `contract_id` and `contract_code_hash` fields are thus an arbitrary projection of the first element.

### Code Paths Examined

- `internal/transform/operation.go:1144-1159` — Both `extend_footprint_ttl` and `restore_footprint` cases call `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` to populate singular `contract_id` and `contract_code_hash` details fields.
- `internal/transform/operation.go:1808-1823` — `contractIdFromTxEnvelope()` iterates `ReadWrite` then `ReadOnly`, returning the first `ContractData` key's contract ID via `contractIdFromContractData()`.
- `internal/transform/operation.go:1826-1839` — `contractIdFromContractData()` extracts contract ID from any `ContractData` ledger key without verifying it is the semantic target.
- `internal/transform/operation.go:1841-1856` — `contractCodeHashFromTxEnvelope()` similarly returns the first `ContractCode` match from `ReadOnly` then `ReadWrite`.
- `internal/transform/operation.go:1859-1873` — `ledgerKeyHashFromTxEnvelope()` returns ALL key hashes as a slice, proving the set-valued nature of the footprint is already recognized elsewhere.

### Findings

1. **Arbitrary first-match selection is confirmed.** Both `contractIdFromTxEnvelope()` and `contractCodeHashFromTxEnvelope()` contain no filtering logic — they accept any matching entry type and return immediately on the first hit.

2. **Distinct from success 024.** Success 024 proved `create_contract` exports the wrong contract ID from the footprint when it should derive it from `ContractIdPreimage`. For `extend_footprint_ttl`/`restore_footprint`, there IS no single canonical contract — these operations apply to an entire footprint set, making the fix fundamentally different (export all, or export none, rather than "derive from preimage").

3. **Multi-contract footprints are realistic.** A single `extend_footprint_ttl` transaction can extend TTL for entries belonging to multiple contracts. A `restore_footprint` can restore entries across contracts. These are valid Soroban transaction patterns.

4. **Inconsistency within the same operation export.** The `ledger_key_hash` field correctly exports all keys as a list, while `contract_id` and `contract_code_hash` export an arbitrary singleton. A downstream consumer comparing these fields would see an inconsistency for multi-contract footprints.

5. **Footprint ordering is not deterministic from the operation's perspective.** The order of keys in `ReadWrite`/`ReadOnly` is determined by transaction construction, not by any semantic priority, so the "first match" varies based on how the transaction was assembled.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Build two distinct contract IDs (contract A and contract B). Construct an `extend_footprint_ttl` operation with a footprint containing `ContractData` keys for both contracts, with contract B appearing first in the `ReadWrite` array.
- **Steps**: Call `TransformOperation()` with the constructed operation and transaction envelope.
- **Assertion**: Assert that `output.OperationDetails["contract_id"]` equals contract B's ID (first in footprint), NOT contract A's. Then swap the footprint order and re-run — the exported `contract_id` should flip, demonstrating order-dependence. Additionally assert that `output.OperationDetails["ledger_key_hash"]` contains hashes for BOTH contracts, proving the underlying data is set-valued while `contract_id` is an arbitrary singleton.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestExtendFootprintTtlArbitraryContractId"
**Test Language**: Go

### Demonstration

The test constructs an `extend_footprint_ttl` operation with two distinct contracts (A and B) in the footprint's ReadWrite array. When contract B is placed first, `contract_id` exports B's address; when the order is swapped so A is first, `contract_id` flips to A's address. Meanwhile, `ledger_key_hash` correctly contains entries for both contracts in both cases, proving the underlying data is set-valued while `contract_id` is an arbitrary order-dependent singleton.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/strkey"
	"github.com/stellar/go-stellar-sdk/xdr"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestExtendFootprintTtlArbitraryContractId demonstrates that extend_footprint_ttl
// exports an arbitrary first-match contract_id when the footprint contains multiple contracts.
func TestExtendFootprintTtlArbitraryContractId(t *testing.T) {
	// Build two distinct contract IDs
	contractIdA := xdr.Hash{0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA,
		0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA, 0xAA}
	contractIdB := xdr.Hash{0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB,
		0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB, 0xBB}

	cidA := xdr.ContractId(contractIdA)
	cidB := xdr.ContractId(contractIdB)

	// Encode the expected strkey addresses for each contract
	contractAddrA, _ := strkey.Encode(strkey.VersionByteContract, contractIdA[:])
	contractAddrB, _ := strkey.Encode(strkey.VersionByteContract, contractIdB[:])

	// Build LedgerKeys for ContractData pointing to each contract.
	// Put contract B first in ReadWrite so it is the "first match".
	ledgerKeyB := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &cidB,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}
	ledgerKeyA := xdr.LedgerKey{
		Type: xdr.LedgerEntryTypeContractData,
		ContractData: &xdr.LedgerKeyContractData{
			Contract: xdr.ScAddress{
				Type:       xdr.ScAddressTypeScAddressTypeContract,
				ContractId: &cidA,
			},
			Key:        xdr.ScVal{Type: xdr.ScValTypeScvLedgerKeyContractInstance},
			Durability: xdr.ContractDataDurabilityPersistent,
		},
	}

	// Build the extend_footprint_ttl operation
	operation := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeExtendFootprintTtl,
			ExtendFootprintTtlOp: &xdr.ExtendFootprintTtlOp{
				Ext:      xdr.ExtensionPoint{V: 0},
				ExtendTo: 5000,
			},
		},
	}

	// Build the envelope with B first, then A in ReadWrite
	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{operation},
			Ext: xdr.TransactionExt{
				V: 0,
				SorobanData: &xdr.SorobanTransactionData{
					Ext:       xdr.SorobanTransactionDataExt{V: 0},
					Resources: xdr.SorobanResources{
						Footprint: xdr.LedgerFootprint{
							ReadWrite: []xdr.LedgerKey{ledgerKeyB, ledgerKeyA}, // B first
							ReadOnly:  []xdr.LedgerKey{},
						},
					},
					ResourceFee: 100,
				},
			},
		},
	}

	// Build the result with ExtendFootprintTtl result type
	opResults := []xdr.OperationResult{
		{
			Code: xdr.OperationResultCodeOpInner,
			Tr: &xdr.OperationResultTr{
				Type: xdr.OperationTypeExtendFootprintTtl,
				ExtendFootprintTtlResult: &xdr.ExtendFootprintTtlResult{
					Code: xdr.ExtendFootprintTtlResultCodeExtendFootprintTtlSuccess,
				},
			},
		},
	}

	txMeta := utils.CreateSampleTxMeta(1, lpAssetA, lpAssetB)
	transaction := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: xdr.TransactionResultPair{
			Result: xdr.TransactionResult{
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: &opResults,
				},
			},
		},
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: txMeta,
		},
	}

	// Run the production transform with B first
	output, err := TransformOperation(operation, 0, transaction, 1, xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{},
		},
	}, "test")
	if err != nil {
		t.Fatalf("TransformOperation failed: %v", err)
	}

	contractIdResult := output.OperationDetails["contract_id"].(string)
	ledgerKeyHashes := output.OperationDetails["ledger_key_hash"].([]string)

	// The exported contract_id should be B (the first in ReadWrite) — demonstrating
	// that the output depends on footprint ordering, not semantic targeting.
	if contractIdResult != contractAddrB {
		t.Errorf("Expected contract_id to be contract B (first in footprint), got %s, want %s",
			contractIdResult, contractAddrB)
	}

	// ledger_key_hash should contain entries for BOTH contracts, proving the
	// underlying data is set-valued while contract_id is a singleton.
	if len(ledgerKeyHashes) < 2 {
		t.Errorf("Expected ledger_key_hash to have at least 2 entries (both contracts), got %d", len(ledgerKeyHashes))
	}

	// Now swap the order: put A first, B second
	envelope2 := envelope
	envelope2.Tx.Ext.SorobanData = &xdr.SorobanTransactionData{
		Ext:       xdr.SorobanTransactionDataExt{V: 0},
		Resources: xdr.SorobanResources{
			Footprint: xdr.LedgerFootprint{
				ReadWrite: []xdr.LedgerKey{ledgerKeyA, ledgerKeyB}, // A first now
				ReadOnly:  []xdr.LedgerKey{},
			},
		},
		ResourceFee: 100,
	}
	envelope2.Tx.Operations = []xdr.Operation{operation}

	transaction2 := transaction
	transaction2.Envelope = xdr.TransactionEnvelope{
		Type: xdr.EnvelopeTypeEnvelopeTypeTx,
		V1:   &envelope2,
	}

	output2, err := TransformOperation(operation, 0, transaction2, 1, xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{},
		},
	}, "test")
	if err != nil {
		t.Fatalf("TransformOperation (swapped order) failed: %v", err)
	}

	contractIdResult2 := output2.OperationDetails["contract_id"].(string)

	// After swapping, contract_id should now be A — proving order-dependence
	if contractIdResult2 != contractAddrA {
		t.Errorf("After swap, expected contract_id to be contract A (now first), got %s, want %s",
			contractIdResult2, contractAddrA)
	}

	// The key proof: the same operation with the same two contracts produces
	// different contract_id outputs depending solely on footprint array ordering.
	if contractIdResult == contractIdResult2 {
		t.Errorf("Expected contract_id to change when footprint order is swapped, but both returned %s", contractIdResult)
	}

	t.Logf("DEMONSTRATED: extend_footprint_ttl contract_id is order-dependent")
	t.Logf("  Order [B, A] -> contract_id = %s", contractIdResult)
	t.Logf("  Order [A, B] -> contract_id = %s", contractIdResult2)
	t.Logf("  ledger_key_hash has %d entries (set-valued), but contract_id is a singleton", len(ledgerKeyHashes))
	t.Logf("  contract A address: %s", contractAddrA)
	t.Logf("  contract B address: %s", contractAddrB)
}
```

### Test Output

```
=== RUN   TestExtendFootprintTtlArbitraryContractId
    data_integrity_poc_test.go:197: DEMONSTRATED: extend_footprint_ttl contract_id is order-dependent
    data_integrity_poc_test.go:198:   Order [B, A] -> contract_id = CC53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53WQD5
    data_integrity_poc_test.go:199:   Order [A, B] -> contract_id = CCVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKUD2U
    data_integrity_poc_test.go:200:   ledger_key_hash has 2 entries (set-valued), but contract_id is a singleton
    data_integrity_poc_test.go:201:   contract A address: CCVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKVKUD2U
    data_integrity_poc_test.go:202:   contract B address: CC53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53XO53WQD5
--- PASS: TestExtendFootprintTtlArbitraryContractId (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.688s
```
