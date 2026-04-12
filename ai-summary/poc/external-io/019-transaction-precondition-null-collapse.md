# H019: Transaction Parquet collapses absent sequence-precondition fields to `0`

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a transaction has no `min_account_sequence`, `min_account_sequence_age`, or `min_account_sequence_ledger_gap` precondition, the Parquet export should preserve that missing state just as the JSON export does. Rows with absent preconditions must remain distinguishable from rows that explicitly set a valid numeric `0` precondition value.

## Mechanism

`TransformTransaction()` models all three sequence-precondition fields as `null.Int` and only sets them when the envelope actually carries the corresponding precondition. `TransactionOutput.ToParquet()` then reads `.Int64` from those nullable wrappers into plain `int64` Parquet columns, so every absent value becomes `0`. That creates a real collision because the repository already treats `0` as a valid concrete value for `min_account_sequence_age` and `min_account_sequence_ledger_gap`.

## Trigger

Run `export_transactions --write-parquet` on any range that contains a transaction without sequence preconditions. Compare the JSON row to the Parquet row, or compare a no-precondition row against a row that explicitly sets `min_account_sequence_age=0` / `min_account_sequence_ledger_gap=0`: JSON preserves the distinction, while Parquet rewrites both to `0`.

## Target Code

- `internal/transform/transaction.go:TransformTransaction:113-129` — builds nullable sequence-precondition fields only when the envelope exposes them
- `internal/transform/schema.go:TransactionOutput:64-67` — models the three fields as `null.Int`
- `internal/transform/schema_parquet.go:TransactionOutputParquet:55-58` — defines the same fields as plain `int64`
- `internal/transform/parquet_converter.go:TransactionOutput.ToParquet:83-86` — flattens invalid `null.Int` values via `.Int64`
- `cmd/export_transactions.go:transactionsCmd.Run:63-65` — writes these lossy Parquet rows during normal exports

## Evidence

The JSON transaction schema explicitly carries `MinAccountSequence`, `MinAccountSequenceAge`, and `MinAccountSequenceLedgerGap` as nullable fields, and `TransformTransaction()` leaves them invalid when the envelope omits those preconditions. The Parquet schema removes that nullability, and the converter reads `.Int64` directly. `internal/transform/transaction_test.go:167-168` already treats `null.IntFrom(0)` as a valid expected output for age and ledger-gap preconditions, so `0` is not a harmless placeholder.

## Anti-Evidence

`min_account_sequence` itself may be less likely to appear as an explicit `0` in live data than the age and ledger-gap fields. But even if only the age/gap fields collide in practice, the Parquet export is still silently converting `null` into a plausible real value.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (same bug class as 013-trade-parquet-null-collapse but distinct fields)

### Trace Summary

`TransformTransaction()` initializes all three precondition fields as `null.Int{}` (zero value: `.Valid=false`, `.Int64=0`) and only populates them via `null.IntFrom()` when the transaction envelope exposes the corresponding precondition. The JSON schema (`TransactionOutput`) preserves nullability through `null.Int`'s JSON marshaling. The Parquet schema (`TransactionOutputParquet`) uses plain `int64`, and `ToParquet()` reads `.Int64` unconditionally, collapsing absent (`null`) values to `0`. The test suite at `transaction_test.go:167-168` uses `null.IntFrom(0)` as a valid expected value, confirming `0` is a legitimate concrete precondition setting.

### Code Paths Examined

- `internal/transform/transaction.go:114,120,126` — `null.Int{}` default initialization for absent preconditions
- `internal/transform/transaction.go:115-117,121-123,127-129` — conditional population via `null.IntFrom()` only when envelope provides values
- `internal/transform/schema.go:65-67` — `null.Int` typed fields in JSON output struct
- `internal/transform/schema_parquet.go:56-58` — plain `int64` typed fields in Parquet output struct (no nullable representation)
- `internal/transform/parquet_converter.go:84-86` — `.Int64` read from `null.Int` without checking `.Valid`
- `internal/transform/transaction_test.go:167-168` — `null.IntFrom(0)` used as valid expected value, proving `0` is a real precondition value

### Findings

The null-to-zero collapse is confirmed for all three transaction precondition fields:
1. **`MinAccountSequence`**: `null.Int{}` → `.Int64` → `0` in Parquet when absent from envelope
2. **`MinAccountSequenceAge`**: Same collapse; test expects `null.IntFrom(0)` as a valid explicit value
3. **`MinAccountSequenceLedgerGap`**: Same collapse; test expects `null.IntFrom(0)` as a valid explicit value

In JSON output, absent preconditions serialize as `null` (via `null.Int`'s marshal behavior), while explicit-zero preconditions serialize as `0`. In Parquet, both become `0`, permanently losing the distinction. This is the same pattern as the confirmed trade-parquet-null-collapse (success/013) but affects a different set of fields.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (or `internal/transform/transaction_test.go` if no converter test exists)
- **Setup**: Create two `TransactionOutput` structs: one with `MinAccountSequenceAge: null.Int{}` (absent) and one with `MinAccountSequenceAge: null.IntFrom(0)` (explicit zero)
- **Steps**: Call `.ToParquet()` on both structs
- **Assertion**: Assert that the resulting `TransactionOutputParquet.MinAccountSequenceAge` values are identical (`0` for both), demonstrating the information loss. The JSON marshaled forms should differ (`null` vs `0`).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTransactionPreconditionNullCollapseInParquet"
**Test Language**: Go

### Demonstration

The test constructs two `TransactionOutput` structs — one with absent preconditions (`null.Int{}`) and one with explicit-zero preconditions (`null.IntFrom(0)`). JSON serialization preserves the distinction (`null` vs `0`), but after `.ToParquet()`, all three precondition fields collapse to `0` in both cases, proving the information loss. The Parquet export permanently destroys the ability to distinguish "no precondition set" from "precondition explicitly set to zero."

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/guregu/null"
)

// TestTransactionPreconditionNullCollapseInParquet demonstrates that
// TransactionOutput.ToParquet() collapses absent (null) precondition fields
// to 0, making them indistinguishable from explicit-zero preconditions.
func TestTransactionPreconditionNullCollapseInParquet(t *testing.T) {
	// 1. Construct two TransactionOutputs:
	//    - "absent" has no preconditions set (null.Int{} zero value, Valid=false)
	//    - "explicitZero" has preconditions explicitly set to 0
	absent := TransactionOutput{
		MinAccountSequence:          null.Int{},        // absent
		MinAccountSequenceAge:       null.Int{},        // absent
		MinAccountSequenceLedgerGap: null.Int{},        // absent
	}

	explicitZero := TransactionOutput{
		MinAccountSequence:          null.IntFrom(0),   // explicit 0
		MinAccountSequenceAge:       null.IntFrom(0),   // explicit 0
		MinAccountSequenceLedgerGap: null.IntFrom(0),   // explicit 0
	}

	// 2. Verify JSON preserves the distinction (null vs 0)
	absentJSON, err := json.Marshal(absent)
	if err != nil {
		t.Fatalf("JSON marshal absent: %v", err)
	}
	explicitJSON, err := json.Marshal(explicitZero)
	if err != nil {
		t.Fatalf("JSON marshal explicit: %v", err)
	}

	// JSON representations should differ: null fields vs 0 fields
	if string(absentJSON) == string(explicitJSON) {
		t.Fatal("JSON representations should differ between absent and explicit-zero preconditions, but they are the same")
	}
	t.Logf("JSON absent:   %s", absentJSON)
	t.Logf("JSON explicit: %s", explicitJSON)

	// 3. Convert both to Parquet and check for information loss
	absentParquet := absent.ToParquet().(TransactionOutputParquet)
	explicitParquet := explicitZero.ToParquet().(TransactionOutputParquet)

	// The bug: Parquet collapses absent (null) to 0, same as explicit zero
	if absentParquet.MinAccountSequence == explicitParquet.MinAccountSequence {
		t.Errorf("MinAccountSequence: Parquet collapsed null to %d, same as explicit zero %d — information lost",
			absentParquet.MinAccountSequence, explicitParquet.MinAccountSequence)
	}

	if absentParquet.MinAccountSequenceAge == explicitParquet.MinAccountSequenceAge {
		t.Errorf("MinAccountSequenceAge: Parquet collapsed null to %d, same as explicit zero %d — information lost",
			absentParquet.MinAccountSequenceAge, explicitParquet.MinAccountSequenceAge)
	}

	if absentParquet.MinAccountSequenceLedgerGap == explicitParquet.MinAccountSequenceLedgerGap {
		t.Errorf("MinAccountSequenceLedgerGap: Parquet collapsed null to %d, same as explicit zero %d — information lost",
			absentParquet.MinAccountSequenceLedgerGap, explicitParquet.MinAccountSequenceLedgerGap)
	}
}
```

### Test Output

```
=== RUN   TestTransactionPreconditionNullCollapseInParquet
    data_integrity_poc_test.go:43: JSON absent:   {"transaction_hash":"","ledger_sequence":0,"account":"","account_sequence":0,"max_fee":0,"fee_charged":0,"operation_count":0,"tx_envelope":"","tx_result":"","tx_meta":"","tx_fee_meta":"","created_at":"0001-01-01T00:00:00Z","memo_type":"","memo":"","time_bounds":"","successful":false,"id":0,"ledger_bounds":"","min_account_sequence":null,"min_account_sequence_age":null,"min_account_sequence_ledger_gap":null,"extra_signers":null,"closed_at":"0001-01-01T00:00:00Z","resource_fee":0,"soroban_resources_instructions":0,"soroban_resources_read_bytes":0,"soroban_resources_disk_read_bytes":0,"soroban_resources_write_bytes":0,"soroban_resources_archived_entries":null,"transaction_result_code":"","inclusion_fee_bid":0,"inclusion_fee_charged":0,"resource_fee_refund":0,"non_refundable_resource_fee_charged":0,"refundable_resource_fee_charged":0,"rent_fee_charged":0,"tx_signers":null}
    data_integrity_poc_test.go:44: JSON explicit: {"transaction_hash":"","ledger_sequence":0,"account":"","account_sequence":0,"max_fee":0,"fee_charged":0,"operation_count":0,"tx_envelope":"","tx_result":"","tx_meta":"","tx_fee_meta":"","created_at":"0001-01-01T00:00:00Z","memo_type":"","memo":"","time_bounds":"","successful":false,"id":0,"ledger_bounds":"","min_account_sequence":0,"min_account_sequence_age":0,"min_account_sequence_ledger_gap":0,"extra_signers":null,"closed_at":"0001-01-01T00:00:00Z","resource_fee":0,"soroban_resources_instructions":0,"soroban_resources_read_bytes":0,"soroban_resources_disk_read_bytes":0,"soroban_resources_write_bytes":0,"soroban_resources_archived_entries":null,"transaction_result_code":"","inclusion_fee_bid":0,"inclusion_fee_charged":0,"resource_fee_refund":0,"non_refundable_resource_fee_charged":0,"refundable_resource_fee_charged":0,"rent_fee_charged":0,"tx_signers":null}
    data_integrity_poc_test.go:52: MinAccountSequence: Parquet collapsed null to 0, same as explicit zero 0 — information lost
    data_integrity_poc_test.go:57: MinAccountSequenceAge: Parquet collapsed null to 0, same as explicit zero 0 — information lost
    data_integrity_poc_test.go:62: MinAccountSequenceLedgerGap: Parquet collapsed null to 0, same as explicit zero 0 — information lost
--- FAIL: TestTransactionPreconditionNullCollapseInParquet (0.00s)
FAIL
```

---

## Final Review — Needs Revision

**Date**: 2026-04-12
**Final review by**: gpt-5.4, high

### What Needs Fixing

The PoC correctly shows that `TransactionOutput.ToParquet()` mechanically flattens `null.Int` to `0`, but the current finding overclaims impact for `min_account_sequence_age` and `min_account_sequence_ledger_gap`. Upstream Stellar code treats zero-valued age/gap preconditions as default no-op values rather than meaningful distinct state: `txnbuild.Preconditions.hasV2Conditions()` only considers those fields when they are `> 0`, and Horizon's resource adapter intentionally omits age `"0"` and tests that ledger-gap `0` should not appear in the rendered preconditions object.

The remaining potentially real issue is narrower: `min_account_sequence` is different, because upstream preserves an explicit `0` when it is actually set (`PopulateTransaction()` emits it whenever the field is valid). The current test does not isolate that narrower case, so it is not sufficient for confirmation as written.

### Revision Instructions

Narrow the hypothesis to `min_account_sequence` unless you can show a concrete downstream contract where age/gap `0` and absent are semantically distinct. Rewrite the PoC to build realistic transactions through `TransformTransaction()`: one transaction with no `MinSeqNum`, and one `PRECOND_V2` transaction with `MinSeqNum` explicitly set to `0`. Then show that JSON/upstream semantics preserve `null` vs `0` for `MinAccountSequence`, while `ToParquet()` collapses both to `0`.

### Checks Passed So Far

- The code path exists: `TransformTransaction()` produces nullable `MinAccountSequence*` fields and `ToParquet()` flattens them with `.Int64`.
- The current PoC reproduces the null-to-zero collapse mechanically.
- The adversarial challenge failed for age/gap because upstream behavior provides a benign explanation: zero is normalized as an empty/default precondition there.

---

## PoC Attempt (Revision)

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTransactionMinAccountSequenceNullCollapseInParquet"
**Test Language**: Go

### Demonstration

The revised test narrows the hypothesis to `MinAccountSequence` only and exercises the full production code path via `TransformTransaction()`. Two realistic `PRECOND_V2` transactions are constructed: one with `MinSeqNum` absent (nil pointer in the XDR `PreconditionsV2`) and one with `MinSeqNum` explicitly set to `0`. `TransformTransaction()` produces `null.Int{}` (Valid=false) for the absent case and `null.IntFrom(0)` for the explicit-zero case. JSON serialization preserves the distinction (`"min_account_sequence":null` vs `"min_account_sequence":0`), but `ToParquet()` collapses both to `int64(0)`, proving the information loss through the production transform pipeline.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTransactionMinAccountSequenceNullCollapseInParquet demonstrates that
// TransformTransaction().ToParquet() collapses absent (null) MinAccountSequence
// to 0, making it indistinguishable from an explicitly-set MinSeqNum of 0 in
// a PRECOND_V2 transaction.
//
// Upstream Stellar semantics distinguish nil MinSeqNum (not set) from *0
// (explicitly set to zero). JSON output preserves this via null.Int marshaling
// (null vs 0). Parquet loses it because ToParquet() reads .Int64 unconditionally.
func TestTransactionMinAccountSequenceNullCollapseInParquet(t *testing.T) {
	hardCodedTransactionHash := xdr.Hash([32]byte{0x01})
	genericResultResults := &[]xdr.OperationResult{
		{
			Tr: &xdr.OperationResultTr{
				Type:                xdr.OperationTypeCreateAccount,
				CreateAccountResult: &xdr.CreateAccountResult{Code: 0},
			},
		},
	}
	meta := xdr.TransactionMeta{
		V:  1,
		V1: genericTxMeta,
	}
	lhe := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			LedgerSeq: 100,
		},
	}

	// Transaction 1: PRECOND_V2 with MinSeqNum absent (nil pointer)
	txAbsent := ingest.LedgerTransaction{
		Index:      1,
		UnsafeMeta: meta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        1,
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondV2,
						V2: &xdr.PreconditionsV2{
							// MinSeqNum is nil — absent
						},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{BumpTo: 1},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	// Transaction 2: PRECOND_V2 with MinSeqNum explicitly set to 0
	explicitZeroSeqNum := xdr.SequenceNumber(0)
	txExplicitZero := ingest.LedgerTransaction{
		Index:      2,
		UnsafeMeta: meta,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1: &xdr.TransactionV1Envelope{
				Tx: xdr.Transaction{
					SourceAccount: testAccount1,
					SeqNum:        1,
					Fee:           100,
					Cond: xdr.Preconditions{
						Type: xdr.PreconditionTypePrecondV2,
						V2: &xdr.PreconditionsV2{
							MinSeqNum: &explicitZeroSeqNum, // explicitly 0
						},
					},
					Operations: []xdr.Operation{
						{
							Body: xdr.OperationBody{
								Type:           xdr.OperationTypeBumpSequence,
								BumpSequenceOp: &xdr.BumpSequenceOp{BumpTo: 1},
							},
						},
					},
				},
			},
		},
		Result: xdr.TransactionResultPair{
			TransactionHash: hardCodedTransactionHash,
			Result: xdr.TransactionResult{
				FeeCharged: 100,
				Result: xdr.TransactionResultResult{
					Code:    xdr.TransactionResultCodeTxSuccess,
					Results: genericResultResults,
				},
			},
		},
	}

	// Run both through production TransformTransaction()
	absentOutput, err := TransformTransaction(txAbsent, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction (absent MinSeqNum): %v", err)
	}
	explicitOutput, err := TransformTransaction(txExplicitZero, lhe)
	if err != nil {
		t.Fatalf("TransformTransaction (explicit-zero MinSeqNum): %v", err)
	}

	// Verify the JSON output preserves the distinction
	type minSeqJSON struct {
		MinAccountSequence *int64 `json:"min_account_sequence"`
	}

	absentJSON, _ := json.Marshal(absentOutput)
	explicitJSON, _ := json.Marshal(explicitOutput)

	var absentParsed, explicitParsed minSeqJSON
	json.Unmarshal(absentJSON, &absentParsed)
	json.Unmarshal(explicitJSON, &explicitParsed)

	if absentParsed.MinAccountSequence != nil {
		t.Errorf("JSON: absent MinAccountSequence should be null, got %d", *absentParsed.MinAccountSequence)
	}
	if explicitParsed.MinAccountSequence == nil || *explicitParsed.MinAccountSequence != 0 {
		t.Errorf("JSON: explicit-zero MinAccountSequence should be 0, got %v", explicitParsed.MinAccountSequence)
	}
	t.Logf("JSON correctly distinguishes absent (null) from explicit zero (0)")

	// Now convert both to Parquet and check for information loss
	absentParquet := absentOutput.ToParquet().(TransactionOutputParquet)
	explicitParquet := explicitOutput.ToParquet().(TransactionOutputParquet)

	// The bug: both become 0 in Parquet, losing the null/zero distinction
	if absentParquet.MinAccountSequence == explicitParquet.MinAccountSequence {
		t.Errorf("MinAccountSequence: Parquet collapsed absent (null) to %d, same as explicit zero %d — information lost",
			absentParquet.MinAccountSequence, explicitParquet.MinAccountSequence)
	}
}
```

### Test Output

```
=== RUN   TestTransactionMinAccountSequenceNullCollapseInParquet
    data_integrity_poc_test.go:148: JSON correctly distinguishes absent (null) from explicit zero (0)
    data_integrity_poc_test.go:156: MinAccountSequence: Parquet collapsed absent (null) to 0, same as explicit zero 0 — information lost
--- FAIL: TestTransactionMinAccountSequenceNullCollapseInParquet (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.665s
FAIL
```
