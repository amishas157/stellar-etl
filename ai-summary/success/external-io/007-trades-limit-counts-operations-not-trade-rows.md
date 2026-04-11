# 007: `export_trades --limit` counts operations, not emitted trade rows

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness / row-limit contract violation
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_trades` documents `--limit` as a maximum number of trades, but the implementation applies that cap to trade-capable operations before `TransformTrade()` expands them into rows. A single successful path-payment or offer-management operation with multiple claimed offers therefore emits multiple trade rows even when only one input item was counted toward the limit.

The original PoC needed correction because its target test file did not exist and it did not directly prove the final working path. After fixing the PoC to use the existing trade fixture and production `TransformTrade()` code, the fan-out is reproducible and the source trace confirms `GetTrades()` enforces the limit at the wrong layer.

## Root Cause

`input.GetTrades()` appends one `TradeTransformInput` per trade-capable operation and stops when the number of collected inputs reaches `limit`. `transform.TransformTrade()` then expands each selected operation into one `TradeOutput` per claimed offer, and `cmd/export_trades.go` writes every returned row without any secondary cap on emitted rows.

## Reproduction

During normal operation, a successful `ManageSellOffer`, `ManageBuyOffer`, `CreatePassiveSellOffer`, or path-payment operation can cross multiple offers and populate multiple `ClaimAtom` entries. When such an operation is included among the first `N` inputs returned by `GetTrades()`, `export_trades --limit N` still writes every emitted row from that operation, exceeding the documented row limit.

## Affected Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — help text promises "`Maximum number of trades to export`"
- `internal/input/trades.go:GetTrades:25-85` — counts one `TradeTransformInput` per qualifying operation and stops on input count
- `internal/transform/trade.go:20-161` — expands one selected operation into one `TradeOutput` per claimed offer
- `cmd/export_trades.go:28-64` — writes every transformed trade row with no row-level limit enforcement

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradesLimitCountsOperationsNotTradeRows`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestTradesLimitCountsOperationsNotTradeRows -v`.

### Test Body

```go
package transform

import (
	"testing"
	"time"
)

func TestTradesLimitCountsOperationsNotTradeRows(t *testing.T) {
	tx := makeTradeTestInput()

	// Operation index 2 is a PathPaymentStrictSend with two claimed offers.
	// GetTrades counts it as one TradeTransformInput, but export_trades writes
	// one row per claimed offer returned by TransformTrade.
	trades, err := TransformTrade(2, 100, tx, time.Unix(0, 0).UTC())
	if err != nil {
		t.Fatalf("TransformTrade returned unexpected error: %v", err)
	}

	if len(trades) != 2 {
		t.Fatalf("expected one trade-producing operation to expand into 2 trade rows, got %d", len(trades))
	}

	if trades[0].HistoryOperationID != trades[1].HistoryOperationID {
		t.Fatalf("expected both rows to come from the same operation, got %d and %d", trades[0].HistoryOperationID, trades[1].HistoryOperationID)
	}

	if trades[0].Order != 0 || trades[1].Order != 1 {
		t.Fatalf("expected output rows to preserve claim order 0 and 1, got %d and %d", trades[0].Order, trades[1].Order)
	}

	const countedInputs = 1
	if len(trades) <= countedInputs {
		t.Fatalf("expected emitted trade rows (%d) to exceed the single counted input operation", len(trades))
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `--limit N` should cap the number of emitted trade rows at `N`, matching the CLI help text.
- **Actual**: `GetTrades()` caps only the number of selected trade-capable operations, so one selected operation can still emit multiple rows.

## Adversarial Review

1. Exercises claimed bug: YES — the fixed PoC uses the production `TransformTrade()` path on a real multi-claim trade fixture and shows one counted operation producing two rows.
2. Realistic preconditions: YES — multi-claim offer crossing is standard Stellar behavior for path payments and offer-management operations.
3. Bug vs by-design: BUG — the shared flag text explicitly promises a maximum number of trades, not a maximum number of candidate operations.
4. Final severity: Medium — rows are individually correct, but the exporter violates its row-bound contract and can return empty, truncated, or oversized results depending on which candidate operations are encountered first.
5. In scope: YES — this is a concrete exporter correctness bug that silently returns plausible but operationally wrong output.
6. Test correctness: CORRECT — the final test proves row fan-out from one counted operation and avoids tautological assertions by checking shared operation identity plus per-row ordering.
7. Alternative explanations: NONE — the only reason more rows appear is that row expansion happens after the limit check.
8. Novelty: NOVEL

## Suggested Fix

Track the number of emitted trade rows in `cmd/export_trades.go` and stop writing once that row count reaches `limit`, including stopping mid-slice when one operation expands into multiple rows. If operation-level limiting is actually intended, change the flag help text and public contract to say so explicitly, but the current implementation does not match the documented row semantics.
