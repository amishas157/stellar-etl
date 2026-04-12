# H001: Open-Ended Ledger Bounds Serialize as a Zero Upper Bound

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a transaction uses `PRECOND_V2.ledgerBounds` with `MaxLedger == 0`, the
exported `ledger_bounds` field should preserve the XDR meaning of "no upper
bound". The row should serialize an open-ended interval, analogous to
`time_bounds`, such as `"[12345,)"`, rather than claiming a real upper ledger
of `0`.

## Mechanism

`TransformTransaction()` formats every ledger-bounds precondition with
`fmt.Sprintf("[%d,%d)", min, max)` and never special-cases the XDR sentinel
`maxLedger == 0`. XDR explicitly defines `0` as "no maxLedger", so the ETL
exports `"[12345,0)"`, which silently changes an open-ended validity window
into an impossible bounded interval.

## Trigger

1. Export a ledger range containing a transaction whose envelope uses
   `PRECOND_V2.ledgerBounds`.
2. Set `ledgerBounds.MinLedger` to a nonzero value and `ledgerBounds.MaxLedger`
   to `0`.
3. Inspect the `history_transactions.ledger_bounds` output for that row.

## Target Code

- `internal/transform/transaction.go:107-110` — serializes `ledger_bounds`
  without handling the `maxLedger == 0` sentinel
- `internal/transform/schema.go:64` — publishes the serialized value as the
  `history_transactions.ledger_bounds` column
- `.../go-stellar-sdk/xdr/Stellar-transaction.x:765-778` — defines
  `maxLedger == 0` as "no maxLedger"

## Evidence

`time_bounds` in the same function already handles its open-ended sentinel by
exporting `"[min,)"`, but `ledger_bounds` does not receive the analogous
treatment. The upstream XDR comment is explicit that `maxLedger == 0` does not
mean ledger zero; it means the upper bound is absent.

## Anti-Evidence

The ETL mirrors the upstream SDK helper
`ingest.LedgerTransaction.LedgerBounds()`, which has the same formatting bug,
so this may have gone unnoticed because the local code follows upstream
behavior. Existing tests only cover bounded intervals like `"[2,20)"`, so
there is no in-repo contract exercising the open-ended case.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `TransformTransaction()` in `internal/transform/transaction.go` lines 90–111. The `time_bounds` block (lines 92–105) explicitly checks `timeBound.MaxTime == 0` and produces the open-ended format `"[%d,)"`. The immediately following `ledger_bounds` block (lines 107–111) has no analogous check — it unconditionally formats `"[%d,%d)"`, emitting `"[12345,0)"` when `MaxLedger == 0`. The XDR spec (`Stellar-transaction.x:741-744`) explicitly defines `maxLedger == 0` as "no maxLedger". Horizon's own `formatLedgerBounds()` correctly treats `MaxLedger == 0` as null via `null.NewInt(int64(ledgerBounds.MaxLedger), ledgerBounds.MaxLedger > 0)`.

### Code Paths Examined

- `internal/transform/transaction.go:92-105` — `time_bounds` correctly handles `MaxTime == 0` with open-ended format `"[%d,)"`
- `internal/transform/transaction.go:107-111` — `ledger_bounds` does NOT handle `MaxLedger == 0`; always uses `"[%d,%d)"` format
- `stellar/go xdr/Stellar-transaction.x:741-744` — XDR comment: `uint32 maxLedger; // 0 here means no maxLedger`
- `stellar/go xdr/Stellar-transaction.x:754-757` — PreconditionsV2 comment: `if maxLedger == 0, then only minLedger is checked`
- `stellar/go services/horizon/internal/db2/history/transaction_ledger_bounds.go:80-85` — Horizon correctly stores `MaxLedger == 0` as null
- `internal/transform/transaction_test.go:433-436` — Test only covers bounded case `{MinLedger: 5, MaxLedger: 10}`
- `internal/transform/schema.go:64` — `LedgerBounds string` field exports the serialized string
- `internal/transform/parquet_converter.go:83` — Parquet passes through the same string directly

### Findings

1. **The bug is confirmed.** `TransformTransaction()` at line 110 formats `ledger_bounds` as `fmt.Sprintf("[%d,%d)", int64(ledgerBound.MinLedger), int64(ledgerBound.MaxLedger))` without checking for the `MaxLedger == 0` sentinel. This produces `"[12345,0)"` for open-ended ledger bounds, which is semantically nonsensical (an interval from 12345 to 0).

2. **The fix pattern already exists in the same function.** Lines 99–103 show exactly how `time_bounds` handles its identical sentinel (`MaxTime == 0`), outputting `"[%d,)"` for open-ended intervals. The ledger bounds code should use the same pattern.

3. **Horizon handles this correctly.** The upstream `formatLedgerBounds()` in `transaction_ledger_bounds.go:80-85` uses `null.NewInt(int64(ledgerBounds.MaxLedger), ledgerBounds.MaxLedger > 0)`, which stores a null MaxLedger when the value is 0, correctly representing "no upper bound."

4. **No test coverage for the open-ended case.** The only test (`transaction_test.go:433-436`) uses `{MinLedger: 5, MaxLedger: 10}`, so the sentinel path was never exercised.

5. **Impact is real.** Any downstream system (BigQuery, analytics) consuming `"[5,0)"` would interpret it as a bounded interval with an impossible upper bound of 0, not as the intended open-ended interval. This affects any `PRECOND_V2` transaction that uses ledger bounds with no upper limit.

### PoC Guidance

- **Test file**: `internal/transform/transaction_test.go`
- **Setup**: Create a test case similar to the existing V2 preconditions test but with `LedgerBounds: &xdr.LedgerBounds{MinLedger: 5, MaxLedger: 0}` (open-ended upper bound)
- **Steps**: Call `TransformTransaction()` with the constructed transaction
- **Assertion**: Assert that `output.LedgerBounds` equals `"[5,)"` (open-ended format matching `time_bounds` behavior), not `"[5,0)"` (the current buggy output)

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestOpenEndedLedgerBoundsSerializeAsZero"
**Test Language**: Go

### Demonstration

The test constructs a transaction with `PrecondV2.LedgerBounds{MinLedger: 5, MaxLedger: 0}` (XDR sentinel for "no upper bound") and calls `TransformTransaction()`. The output `LedgerBounds` field is `"[5,0)"` instead of the expected `"[5,)"`, proving that the ETL serializes the open-ended sentinel as a literal zero upper bound. This contrasts with `time_bounds` handling in the same function, which correctly outputs `"[%d,)"` when `MaxTime == 0`.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/utils"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestOpenEndedLedgerBoundsSerializeAsZero demonstrates that when a transaction
// has PrecondV2 ledger bounds with MaxLedger == 0 (meaning "no upper bound"
// per the XDR spec), TransformTransaction incorrectly serializes the bounds
// as "[5,0)" instead of the open-ended format "[5,)".
func TestOpenEndedLedgerBoundsSerializeAsZero(t *testing.T) {
	// 1. Construct a transaction with open-ended ledger bounds (MaxLedger == 0)
	envelope := xdr.TransactionV1Envelope{
		Tx: xdr.Transaction{
			SourceAccount: genericSourceAccount,
			SeqNum:        1,
			Fee:           100,
			Memo:          xdr.Memo{Type: xdr.MemoTypeMemoNone},
			Cond: xdr.Preconditions{
				Type: xdr.PreconditionTypePrecondV2,
				V2: &xdr.PreconditionsV2{
					TimeBounds: &xdr.TimeBounds{
						MinTime: 0,
						MaxTime: 1594272628,
					},
					LedgerBounds: &xdr.LedgerBounds{
						MinLedger: 5,
						MaxLedger: 0, // XDR sentinel: means "no upper bound"
					},
				},
			},
			Operations: []xdr.Operation{
				genericBumpOperationForTransaction,
			},
		},
	}

	tx := ingest.LedgerTransaction{
		Index: 1,
		Envelope: xdr.TransactionEnvelope{
			Type: xdr.EnvelopeTypeEnvelopeTypeTx,
			V1:   &envelope,
		},
		Result:   utils.CreateSampleResultMeta(true, 10).Result,
		UnsafeMeta: xdr.TransactionMeta{
			V:  1,
			V1: genericTxMeta,
		},
	}

	header := xdr.LedgerHeaderHistoryEntry{}

	// 2. Run production code
	output, err := TransformTransaction(tx, header)
	if err != nil {
		t.Fatalf("TransformTransaction returned unexpected error: %v", err)
	}

	// 3. Assert the output preserves the XDR semantics of open-ended bounds.
	//    The time_bounds code already uses "[%d,)" for MaxTime == 0.
	//    Ledger bounds should do the same for MaxLedger == 0.
	expected := "[5,)"
	if output.LedgerBounds != expected {
		t.Errorf("LedgerBounds corrupted: got %q, want %q (MaxLedger==0 means no upper bound per XDR spec)", output.LedgerBounds, expected)
	}
}
```

### Test Output

```
=== RUN   TestOpenEndedLedgerBoundsSerializeAsZero
    data_integrity_poc_test.go:69: LedgerBounds corrupted: got "[5,0)", want "[5,)" (MaxLedger==0 means no upper bound per XDR spec)
--- FAIL: TestOpenEndedLedgerBoundsSerializeAsZero (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.822s
```
