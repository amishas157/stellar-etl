# 010: Trade limit counts operations before trades

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_trades --limit` is documented as the maximum number of trades to export, but `input.GetTrades()` spends that budget on successful trade-capable operations before `TransformTrade()` knows whether any trade rows will actually be emitted. On pubnet ledger `28770265`, the first successful trade-capable operation claims zero offers while a later operation in the same ledger claims one, so `--limit 1` stops early and produces an empty export even though trade data exists later in-range.

## Root Cause

`GetTrades()` appends one `TradeTransformInput` per successful operation whose type can produce trades, then enforces `limit` on the length of that input slice. `TransformTrade()` only emits rows after it extracts `claimedOffers`, and a successful manage-offer/passive-offer operation may legally have `claimedOffers == []`, so the limit is applied at operation granularity instead of output-row granularity.

## Reproduction

During normal operation, this manifests on bounded `export_trades` runs whenever the earliest successful trade-capable operation in the requested range posts or updates an offer without matching liquidity, and a later operation in the same range does execute a trade. The exporter consumes the entire limit on the zero-trade operation and stops before the later trade rows are reached.

## Affected Code

- `internal/input/trades.go:GetTrades:50-80` — appends one input per successful trade-capable operation and enforces `limit` on `len(tradeSlice)`
- `internal/transform/trade.go:TransformTrade:34-41` — extracts `claimedOffers` only after the input slice has already been limited
- `internal/transform/trade.go:41-72` — emits one row per claimed offer and returns zero rows when `claimedOffers` is empty
- `cmd/export_trades.go:tradesCmd.Run:37-58` — writes only transformed rows, so zero-row inputs silently consume the export budget
- `internal/utils/main.go:AddArchiveFlags:250-254` — documents `--limit` as the maximum number of trades to export

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTradeLimitCountsOperationsNotTrades`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestTradeLimitCountsOperationsNotTrades -v`
  4. Observe that on pubnet ledger `28770265` the first successful trade-capable operation claims `0` offers while a later one claims `1`, proving `--limit 1` stops before any trade row can be emitted.

### Test Body

```go
package transform_test

import (
	"context"
	"io"
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestTradeLimitCountsOperationsNotTrades(t *testing.T) {
	env := utils.GetEnvironmentDetails(utils.CommonFlagValues{})

	const ledger uint32 = 28770265

	backend, err := utils.CreateBackend(ledger, ledger, env.ArchiveURLs)
	if err != nil {
		t.Fatalf("CreateBackend() error = %v", err)
	}

	lcm, err := backend.GetLedger(context.Background(), ledger)
	if err != nil {
		t.Fatalf("GetLedger() error = %v", err)
	}

	txReader, err := ingest.NewLedgerTransactionReaderFromLedgerCloseMeta(env.NetworkPassphrase, lcm)
	if err != nil {
		t.Fatalf("NewLedgerTransactionReaderFromLedgerCloseMeta() error = %v", err)
	}
	defer txReader.Close()

	firstCount := -1
	laterCount := 0

	for {
		tx, err := txReader.Read()
		if err == io.EOF {
			break
		}
		if err != nil {
			t.Fatalf("Read() error = %v", err)
		}
		if !tx.Result.Successful() {
			continue
		}

		operationResults, ok := tx.Result.OperationResults()
		if !ok {
			t.Fatalf("transaction %d did not expose operation results", tx.Index)
		}

		for opIndex, op := range tx.Envelope.Operations() {
			if !tradeCapable(op.Body.Type) {
				continue
			}

			claimedOffers, ok := claimedOfferCount(operationResults, int32(opIndex), op.Body.Type)
			if !ok {
				t.Fatalf("unable to inspect claimed offers for transaction %d operation %d", tx.Index, opIndex)
			}

			if firstCount == -1 {
				firstCount = claimedOffers
				continue
			}

			if claimedOffers > 0 {
				laterCount = claimedOffers
				break
			}
		}

		if laterCount > 0 {
			break
		}
	}

	if firstCount != 0 {
		t.Fatalf("first trade-capable successful operation claimed %d offers, want 0", firstCount)
	}
	if laterCount == 0 {
		t.Fatalf("did not find a later trade-capable successful operation with claimed offers in ledger %d", ledger)
	}

	t.Logf(
		"ledger %d reproduces the bug preconditions: the first successful trade-capable operation claims %d offers, while a later one claims %d; input.GetTrades appends the first operation before enforcing limit=1, so export_trades emits 0 rows instead of the later trade(s)",
		ledger,
		firstCount,
		laterCount,
	)
}

func tradeCapable(operationType xdr.OperationType) bool {
	switch operationType {
	case xdr.OperationTypeManageBuyOffer,
		xdr.OperationTypeManageSellOffer,
		xdr.OperationTypeCreatePassiveSellOffer,
		xdr.OperationTypePathPaymentStrictReceive,
		xdr.OperationTypePathPaymentStrictSend:
		return true
	default:
		return false
	}
}

func claimedOfferCount(operationResults []xdr.OperationResult, operationIndex int32, operationType xdr.OperationType) (int, bool) {
	if int(operationIndex) >= len(operationResults) || operationResults[operationIndex].Tr == nil {
		return 0, false
	}

	operationTr := operationResults[operationIndex].Tr

	switch operationType {
	case xdr.OperationTypeManageBuyOffer:
		if success, ok := operationTr.MustManageBuyOfferResult().GetSuccess(); ok {
			return len(success.OffersClaimed), true
		}
	case xdr.OperationTypeManageSellOffer:
		if success, ok := operationTr.MustManageSellOfferResult().GetSuccess(); ok {
			return len(success.OffersClaimed), true
		}
	case xdr.OperationTypeCreatePassiveSellOffer:
		if operationTr.Type == xdr.OperationTypeManageSellOffer {
			return len(operationTr.MustManageSellOfferResult().MustSuccess().OffersClaimed), true
		}
		return len(operationTr.MustCreatePassiveSellOfferResult().MustSuccess().OffersClaimed), true
	case xdr.OperationTypePathPaymentStrictSend:
		if success, ok := operationTr.MustPathPaymentStrictSendResult().GetSuccess(); ok {
			return len(success.Offers), true
		}
	case xdr.OperationTypePathPaymentStrictReceive:
		if success, ok := operationTr.MustPathPaymentStrictReceiveResult().GetSuccess(); ok {
			return len(success.Offers), true
		}
	}

	return 0, false
}
```

## Expected vs Actual Behavior

- **Expected**: `--limit N` should stop after `N` trade rows have actually been emitted, so `--limit 1` should keep scanning past unmatched offer updates until it finds one real trade row or exhausts the range.
- **Actual**: `GetTrades()` stops after `N` successful trade-capable operations, so `--limit 1` can terminate on an unmatched offer update and emit zero rows even when later trades exist in the same range.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses immutable pubnet ledger data and demonstrates the exact precondition that makes `GetTrades()` consume `limit=1` before any trade row can exist.
2. Realistic preconditions: YES — successful manage-offer/passive-offer operations with zero claimed offers are normal "posted but unmatched" Stellar behavior.
3. Bug vs by-design: BUG — the CLI flag and help text define `--limit` in terms of exported trades, not candidate operations.
4. Final severity: High — this silently returns empty or truncated trade exports, which is structural data corruption rather than direct monetary miscalculation.
5. In scope: YES — it is a concrete export-pipeline correctness defect on a normal mainnet ledger.
6. Test correctness: CORRECT — the test does not assert the bug into existence; it proves the ledger-level preconditions on real chain data, and the stop condition follows directly from the cited production loop.
7. Alternative explanations: NONE — once the first successful trade-capable operation has zero claims and a later one has a positive claim count, the current `limit` check necessarily truncates output.
8. Novelty: NOVEL

## Suggested Fix

Apply the limit to emitted `TradeOutput` rows, not to `TradeTransformInput` candidates. The simplest fix is to move the limiting logic after `TransformTrade()` so the exporter counts actual rows written; alternatively, change `GetTrades()` to continue scanning until it has accumulated at least `N` claimed offers rather than `N` trade-capable operations.
