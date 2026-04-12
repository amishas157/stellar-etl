# 019: Open-Ended Ledger Bounds Serialize as Zero Upper Bound

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformTransaction()` exports `PRECOND_V2.ledgerBounds` as a string in `history_transactions.ledger_bounds`. When XDR uses `MaxLedger == 0` to mean "no upper bound", the ETL instead emits `"[min,0)"`, silently converting an open-ended validity window into an impossible bounded interval.

## Root Cause

`internal/transform/transaction.go` formats every non-nil ledger-bounds precondition with `fmt.Sprintf("[%d,%d)", min, max)` and never handles the XDR sentinel `MaxLedger == 0`. The bad string is then published unchanged in JSON and carried straight through the Parquet converter.

## Reproduction

Any normal export that includes a transaction with `PreconditionsV2.LedgerBounds{MinLedger: 5, MaxLedger: 0}` will hit this path. The transformed row reports `ledger_bounds` as `"[5,0)"` even though the XDR contract defines that precondition as open-ended and equivalent to `"[5,)"`.

## Affected Code

- `internal/transform/transaction.go:TransformTransaction:107-110` — serializes `ledger_bounds` without handling the `MaxLedger == 0` sentinel
- `internal/transform/schema.go:TransactionOutput:64` — exposes the corrupted value as the exported `ledger_bounds` column
- `internal/transform/parquet_converter.go:ConvertToTransactionParquet:83` — propagates the corrupted string into Parquet output

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestOpenEndedLedgerBoundsSerializeAsZero`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run TestOpenEndedLedgerBoundsSerializeAsZero -v` after creating the target test file with the body below.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestOpenEndedLedgerBoundsSerializeAsZero(t *testing.T) {
	tx := genericLedgerTransaction
	envelope := genericBumpOperationEnvelopeForTransaction
	envelope.Tx.Cond = xdr.Preconditions{
		Type: xdr.PreconditionTypePrecondV2,
		V2: &xdr.PreconditionsV2{
			LedgerBounds: &xdr.LedgerBounds{
				MinLedger: 5,
				MaxLedger: 0,
			},
		},
	}
	tx.Envelope.V1 = &envelope

	output, err := TransformTransaction(tx, genericLedgerHeaderHistoryEntry)
	if err != nil {
		t.Fatalf("TransformTransaction() error = %v", err)
	}

	if output.LedgerBounds != "[5,)" {
		t.Fatalf("LedgerBounds = %q, want %q", output.LedgerBounds, "[5,)")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `ledger_bounds` preserves the XDR meaning of `MaxLedger == 0` and exports an open-ended interval such as `"[5,)"`
- **Actual**: `ledger_bounds` is exported as `"[5,0)"`, which claims a literal upper ledger of `0`

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production `TransformTransaction()` path and inspects the exported `LedgerBounds` field.
2. Realistic preconditions: YES — `PRECOND_V2` ledger bounds are standard transaction preconditions, and XDR explicitly defines `MaxLedger == 0` as "no maxLedger".
3. Bug vs by-design: BUG — the same function already renders open-ended `time_bounds` as `"[min,)"`, and Horizon's `formatLedgerBounds()` also elides a zero upper bound.
4. Final severity: High — this corrupts a non-financial exported validity-window field that downstream consumers can parse as real bounded data.
5. In scope: YES — the ETL emits wrong persisted output without crashing or surfacing an error.
6. Test correctness: CORRECT — the test uses real package fixtures, no mocks, and the assertion is tied to the XDR contract rather than to a self-fulfilling expectation.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Treat `ledgerBound.MaxLedger == 0` the same way `time_bounds` treats `MaxTime == 0`: emit `fmt.Sprintf("[%d,)", ledgerBound.MinLedger)`. Add a regression test for the open-ended case and consider mirroring the existing `time_bounds` validation for invalid `max < min` ledger bounds.
