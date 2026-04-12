# H001: `upload_wasm` operation details can report the wrong `contract_code_hash`

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural identifier corruption in `history_operations.details`
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `history_operations` rows where `type=invoke_host_function` and
`details.type="upload_wasm"`, the exported `details.contract_code_hash` should
identify the WASM blob carried by that operation's host-function payload. If the
same transaction footprint also mentions other `ContractCode` ledger keys, those
extra keys should not change the reported hash for the uploaded code.

## Mechanism

The live `upload_wasm` branch ignores the host function's raw `Wasm` bytes and
fills `details.contract_code_hash` by calling `contractCodeHashFromTxEnvelope()`.
That helper scans `ReadOnly` first and `ReadWrite` second and returns the first
`LedgerKeyContractCode.Hash` it encounters, so the exported hash is chosen by
footprint ordering rather than by the uploaded payload. A transaction whose
footprint contains another contract-code key ahead of the newly uploaded code can
therefore export a syntactically valid but wrong `contract_code_hash`.

## Trigger

1. Build a Soroban `upload_contract_wasm` transaction whose `HostFunction.Wasm`
   payload hashes to code hash `A`.
2. Include another `LedgerKeyContractCode` with hash `B` earlier in the declared
   footprint than the uploaded code's key.
3. Export operations and inspect the `upload_wasm` row: `details.contract_code_hash`
   will follow the first footprint code key (`B`) instead of the uploaded WASM's
   hash (`A`).

## Target Code

- `internal/transform/operation.go:1114-1118` — `upload_wasm` branch sources `contract_code_hash` from the footprint helper
- `internal/transform/operation.go:1841-1857` — helper returns the first `ContractCode` key found in `ReadOnly`/`ReadWrite`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:29175-29181` — `HostFunctionTypeUploadContractWasm` stores raw WASM bytes in `HostFunction.Wasm`

## Evidence

The operation branch has direct access to the host function but never reads the
`Wasm` payload at all. Instead it delegates to a transaction-scoped footprint
scanner that is explicitly order-dependent (`ReadOnly` before `ReadWrite`,
first match wins), which is a poor fit for an operation whose authoritative code
artifact is already present in the operation body.

## Anti-Evidence

Many `upload_wasm` transactions may still export the correct hash in practice if
their footprint contains only the newly created `ContractCode` key. The existing
unit tests also use generic envelopes with no Soroban footprint and therefore
assert empty hashes rather than exercising competing `ContractCode` entries.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Fail entry 044 covered `extend_footprint_ttl`/`restore_footprint` first-match behavior (dismissed as intentionally lossy summaries), but `upload_wasm` is distinct because it has an authoritative source — the WASM bytes in the operation body — making the footprint-based approach a genuine correctness bug rather than a design trade-off.

### Trace Summary

The `upload_wasm` case in `TransformOperation` (line 1114) and its method-receiver sibling (line 1736) both call `contractCodeHashFromTxEnvelope()` to populate `details["contract_code_hash"]`. This helper (lines 1841-1857) iterates the transaction footprint's `ReadOnly` keys first, then `ReadWrite` keys, returning the hex hash of the first `LedgerKeyContractCode` encountered. The uploaded WASM's code key would normally reside in `ReadWrite` (it's being created), so any `ContractCode` key in `ReadOnly` would win. The `HostFunction.MustWasm()` accessor (XDR line 29243) provides the raw WASM bytes — `sha256(wasm)` is the authoritative hash — but neither code path ever reads it.

### Code Paths Examined

- `internal/transform/operation.go:1114-1118` — `upload_wasm` case in free-function `TransformOperation`: sets `details["contract_code_hash"]` via `contractCodeHashFromTxEnvelope(transactionEnvelope)`, never accesses `op.HostFunction.MustWasm()`
- `internal/transform/operation.go:1736-1740` — `upload_wasm` case in method-receiver variant: identical pattern, same footprint-based helper, same omission of WASM bytes
- `internal/transform/operation.go:1841-1857` — `contractCodeHashFromTxEnvelope()`: scans `ReadOnly` then `ReadWrite`, returns first `ContractCode` hash via `contractCodeFromContractData()`
- `internal/transform/operation.go:1876-1884` — `contractCodeFromContractData()`: extracts `Hash.HexString()` from a `LedgerKeyContractCode`
- XDR `HostFunction.MustWasm()` (line 29243-29248): returns raw `[]byte` WASM for `UploadContractWasm` host function type — confirmed available but unused

### Findings

1. **Two affected code paths**: Both the free-function `TransformOperation` (line 1114) and the method-receiver version (line 1736) exhibit the identical bug. Both delegate to `contractCodeHashFromTxEnvelope` without considering the WASM payload.

2. **ReadOnly wins over ReadWrite**: The helper scans `ReadOnly` before `ReadWrite`. For `upload_wasm`, the newly uploaded code's key resides in `ReadWrite` (it's being created). If any `ContractCode` key exists in `ReadOnly` (e.g., referencing pre-existing contract code), it takes precedence, producing the wrong hash.

3. **Authoritative source ignored**: `op.HostFunction.MustWasm()` provides the raw WASM bytes. `sha256(wasm_bytes)` equals the canonical `ContractCode` hash. This is the definitive, operation-specific source. The footprint scanner is a transaction-scoped heuristic that conflates all code keys.

4. **Distinguishable from fail entry 044**: The failed hypothesis 044 concerned `extend_footprint_ttl`/`restore_footprint`, where no single authoritative target exists and the scalar is an intentionally lossy summary. For `upload_wasm`, the operation body carries the exact artifact being uploaded, making the first-match approach a data correctness bug.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (append new test case)
- **Setup**: Construct a `xdr.TransactionEnvelope` with Soroban data containing:
  - `ReadOnly` footprint with one `LedgerKeyContractCode` entry whose hash is `B` (an unrelated pre-existing code hash)
  - `ReadWrite` footprint with one `LedgerKeyContractCode` entry whose hash is `A` (the hash of the uploaded WASM)
  - A `HostFunction` of type `HostFunctionTypeUploadContractWasm` with WASM bytes whose SHA-256 equals `A`
- **Steps**: Call `TransformOperation` with the constructed operation and transaction
- **Assertion**: Assert that `details["contract_code_hash"]` equals `A` (the uploaded WASM's hash). Currently it will equal `B` (the first `ReadOnly` code key), demonstrating the bug.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestUploadWasmContractCodeHashFollowsFootprintOrder"
**Test Language**: Go

### Demonstration

The test constructs a Soroban `upload_wasm` transaction with WASM bytes whose SHA-256 is hash `A`, places an unrelated `ContractCode` key with hash `B` in the `ReadOnly` footprint, and the uploaded code's key with hash `A` in the `ReadWrite` footprint. When `TransformOperation` is called, `details["contract_code_hash"]` returns `B` (the ReadOnly key) instead of `A` (the uploaded WASM's hash), confirming that the exported hash follows footprint iteration order rather than the authoritative WASM payload.

### Test Body

```go
package transform

import (
	"crypto/sha256"
	"encoding/hex"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"

	"github.com/stellar/stellar-etl/v2/internal/utils"
)

// TestUploadWasmContractCodeHashFollowsFootprintOrder demonstrates that
// upload_wasm operations report the wrong contract_code_hash when another
// ContractCode key appears earlier in the transaction footprint.
//
// The authoritative hash should be sha256(WASM bytes) from the operation body,
// but contractCodeHashFromTxEnvelope returns the first ContractCode key it
// encounters scanning ReadOnly before ReadWrite. When an unrelated ContractCode
// key sits in ReadOnly, it wins — producing a wrong hash.
func TestUploadWasmContractCodeHashFollowsFootprintOrder(t *testing.T) {
	// 1. Build WASM bytes and compute the authoritative hash (A)
	wasmBytes := []byte("sample-wasm-payload-for-poc-test")
	wasmSHA := sha256.Sum256(wasmBytes)
	hashA := hex.EncodeToString(wasmSHA[:]) // the correct hash of the uploaded code

	// 2. Create an unrelated contract code hash (B), different from A
	var hashBBytes xdr.Hash
	for i := range hashBBytes {
		hashBBytes[i] = 0xBB // arbitrary, clearly different from wasmSHA
	}
	hashB := hex.EncodeToString(hashBBytes[:])

	// Sanity: the two hashes must differ
	if hashA == hashB {
		t.Fatal("test setup error: hashA and hashB should differ")
	}

	// 3. Build the footprint: ReadOnly has hash B, ReadWrite has hash A
	var hashAXdr xdr.Hash
	copy(hashAXdr[:], wasmSHA[:])

	footprint := xdr.LedgerFootprint{
		ReadOnly: []xdr.LedgerKey{
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: hashBBytes, // unrelated code — appears first
				},
			},
		},
		ReadWrite: []xdr.LedgerKey{
			{
				Type: xdr.LedgerEntryTypeContractCode,
				ContractCode: &xdr.LedgerKeyContractCode{
					Hash: hashAXdr, // the uploaded code's hash
				},
			},
		},
	}

	// 4. Build the TransactionV1Envelope with Soroban data and the upload_wasm operation
	uploadWasmOp := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeInvokeHostFunction,
			InvokeHostFunctionOp: &xdr.InvokeHostFunctionOp{
				HostFunction: xdr.HostFunction{
					Type: xdr.HostFunctionTypeHostFunctionTypeUploadContractWasm,
					Wasm: &wasmBytes,
				},
			},
		},
	}

	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			Memo:          xdr.Memo{},
			Operations:    []xdr.Operation{uploadWasmOp},
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

	// 5. Build a LedgerTransaction
	ledgerTx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result: utils.CreateSampleResultMeta(true, 10).Result,
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: genericTxMeta,
		},
	}

	ledgerCloseMeta := makeLedgerCloseMeta()

	// 6. Call TransformOperation
	output, err := TransformOperation(
		uploadWasmOp,
		0,
		ledgerTx,
		1, // ledgerSeq
		ledgerCloseMeta,
		"", // network
	)
	if err != nil {
		t.Fatalf("TransformOperation returned unexpected error: %v", err)
	}

	// 7. Verify the bug: contract_code_hash should be hashA (the uploaded WASM's hash)
	// but the current code returns hashB (the first ContractCode key in ReadOnly).
	gotHash, ok := output.OperationDetails["contract_code_hash"].(string)
	if !ok {
		t.Fatalf("contract_code_hash not found or not a string in details: %v", output.OperationDetails)
	}

	// The authoritative hash of the uploaded WASM
	t.Logf("Uploaded WASM sha256 (hashA): %s", hashA)
	t.Logf("Unrelated ReadOnly code key  (hashB): %s", hashB)
	t.Logf("Exported contract_code_hash:          %s", gotHash)

	if gotHash == hashB {
		t.Errorf("BUG CONFIRMED: contract_code_hash follows footprint order.\n"+
			"  Got:  %s (hashB — from ReadOnly footprint, unrelated code)\n"+
			"  Want: %s (hashA — sha256 of the uploaded WASM bytes)", gotHash, hashA)
	} else if gotHash != hashA {
		t.Errorf("contract_code_hash is neither hashA nor hashB.\n"+
			"  Got:  %s\n"+
			"  hashA (expected): %s\n"+
			"  hashB (footprint): %s", gotHash, hashA, hashB)
	} else {
		t.Logf("contract_code_hash correctly matches uploaded WASM hash (hashA)")
	}
}
```

### Test Output

```
=== RUN   TestUploadWasmContractCodeHashFollowsFootprintOrder
    data_integrity_poc_test.go:132: Uploaded WASM sha256 (hashA): a377fd08b852879247907c7ffb6a3268bd524ecc7576461327e61e3f032d40b3
    data_integrity_poc_test.go:133: Unrelated ReadOnly code key  (hashB): bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:134: Exported contract_code_hash:          bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
    data_integrity_poc_test.go:137: BUG CONFIRMED: contract_code_hash follows footprint order.
          Got:  bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb (hashB — from ReadOnly footprint, unrelated code)
          Want: a377fd08b852879247907c7ffb6a3268bd524ecc7576461327e61e3f032d40b3 (hashA — sha256 of the uploaded WASM bytes)
--- FAIL: TestUploadWasmContractCodeHashFollowsFootprintOrder (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.852s
```
