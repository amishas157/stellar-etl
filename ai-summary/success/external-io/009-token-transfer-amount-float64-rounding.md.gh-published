# 009: Token transfer amount float64 rounding

**Date**: 2026-04-11
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`transformEvents()` converts raw token-transfer amount strings into `float64` with `strconv.ParseFloat` and then scales them by `1e-7` before export. For raw values above the 53-bit float mantissa limit, distinct on-chain amounts collapse to the same exported JSON `amount` even though `amount_raw` still preserves the exact source value.

The issue is reproducible through the production transform path with a realistic `TokenTransferEvent`: raw amounts `9007199254740992` and `9007199254740993` both export as `900719925.4740992`. That is silent monetary data corruption in a first-class numeric field.

## Root Cause

The token-transfer exporter treats arbitrary-precision decimal strings as if they were safely representable in `float64`. `strconv.ParseFloat(..., 64)` rounds integers above `2^53`, and the already-rounded value is then scaled and stored in `TokenTransferOutput.Amount`.

## Reproduction

During normal token-transfer export, the upstream SDK emits event amounts as decimal strings derived from `i128` values. When `transformEvents()` handles a transfer, mint, burn, clawback, or fee event whose raw amount exceeds `2^53`, the ETL rounds the value during `ParseFloat`, emits the rounded `amount` to JSON, and keeps the exact source only in `amount_raw`.

## Affected Code

- `internal/transform/token_transfer.go:37-53` — parses transfer-style event amounts with `strconv.ParseFloat` and scales them by `1e-7`
- `internal/transform/token_transfer.go:54-73` — repeats the same lossy conversion for mint, burn, clawback, and fee events
- `internal/transform/token_transfer.go:108-123` — exports the rounded float64 as `TokenTransferOutput.Amount`
- `internal/transform/schema.go:659-671` — publishes `amount` as a numeric JSON field alongside exact `amount_raw`

## PoC

- **Target test file**: `internal/transform/token_transfer_float64_poc_test.go`
- **Test name**: `TestTokenTransferAmountFloat64Rounding`
- **Test language**: go
- **How to run**: Create the test file below, then run `go test ./internal/transform/... -run TestTokenTransferAmountFloat64Rounding -v`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"math"
	"math/big"
	"strings"
	"testing"

	"github.com/stellar/go-stellar-sdk/asset"
	"github.com/stellar/go-stellar-sdk/processors/token_transfer"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTokenTransferAmountFloat64Rounding(t *testing.T) {
	ledger := xdr.LedgerCloseMeta{
		V: 1,
		V1: &xdr.LedgerCloseMetaV1{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue: xdr.StellarValue{CloseTime: 1000},
					LedgerSeq: 10,
				},
			},
		},
	}

	outA := singleTokenTransferOutput(t, ledger, "9007199254740992")
	outB := singleTokenTransferOutput(t, ledger, "9007199254740993")

	if outA.AmountRaw == outB.AmountRaw {
		t.Fatalf("precondition failed: raw amounts should differ")
	}

	if math.Float64bits(outA.Amount) != math.Float64bits(outB.Amount) {
		t.Fatalf("expected float64 collision, got %0.10f and %0.10f", outA.Amount, outB.Amount)
	}

	js, err := json.Marshal(outB)
	if err != nil {
		t.Fatalf("marshal output: %v", err)
	}

	const exactScaled = "900719925.4740993"
	if strings.Contains(string(js), `"amount":`+exactScaled) {
		t.Fatalf("expected exported JSON amount to lose precision, got %s", js)
	}

	if !strings.Contains(string(js), `"amount":900719925.4740992`) {
		t.Fatalf("expected rounded JSON amount, got %s", js)
	}

	if !strings.Contains(string(js), `"amount_raw":"9007199254740993"`) {
		t.Fatalf("expected exact raw amount in JSON, got %s", js)
	}

	wantExact := new(big.Rat)
	if _, ok := wantExact.SetString("9007199254740993/10000000"); !ok {
		t.Fatal("failed to build exact rational amount")
	}

	if got := wantExact.FloatString(7); got != exactScaled {
		t.Fatalf("test precondition failed: exact scaled amount = %s, want %s", got, exactScaled)
	}
}

func singleTokenTransferOutput(t *testing.T, ledger xdr.LedgerCloseMeta, amount string) TokenTransferOutput {
	t.Helper()

	operationIndex := uint32(1)
	events := []*token_transfer.TokenTransferEvent{
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash",
				TransactionIndex: 1,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contractaddress",
			},
			Event: &token_transfer.TokenTransferEvent_Transfer{
				Transfer: &token_transfer.Transfer{
					From: "from",
					To:   "to",
					Asset: &asset.Asset{
						AssetType: &asset.Asset_IssuedAsset{
							IssuedAsset: &asset.IssuedAsset{
								AssetCode: "abc",
								Issuer:    "def",
							},
						},
					},
					Amount: amount,
				},
			},
		},
	}

	outputs, err := transformEvents(events, ledger)
	if err != nil {
		t.Fatalf("transformEvents returned error: %v", err)
	}

	if len(outputs) != 1 {
		t.Fatalf("expected 1 output, got %d", len(outputs))
	}

	return outputs[0]
}
```

## Expected vs Actual Behavior

- **Expected**: distinct `amount_raw` values should produce distinct exact exported `amount` values after the `1e-7` scale factor is applied
- **Actual**: raw values `9007199254740992` and `9007199254740993` both export the same JSON number, `900719925.4740992`, while `amount_raw` remains distinct

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls `transformEvents()` directly and verifies the exported JSON emitted from `TokenTransferOutput`
2. Realistic preconditions: YES — the upstream SDK supplies token-transfer amounts as decimal strings extracted from `i128` values, and values above `2^53` are representable on-chain
3. Bug vs by-design: BUG — the exporter publishes `amount` as a first-class numeric field without documenting it as approximate, while simultaneously preserving the exact source in `amount_raw`
4. Final severity: Critical — the corrupted field is a monetary amount consumed as a numeric export field
5. In scope: YES — this is silent data corruption in exported blockchain data
6. Test correctness: CORRECT — the test uses the production transform path, proves a collision between two distinct raw values, and inspects the actual marshaled JSON output
7. Alternative explanations: NONE
8. Novelty: Unassessed here — duplicate handling is delegated to the orchestrator

## Suggested Fix

Stop deriving token-transfer `amount` through `float64`. Either export the exact scaled decimal as a string/decimal type, or compute it with arbitrary-precision arithmetic and serialize it without passing through binary floating-point.
