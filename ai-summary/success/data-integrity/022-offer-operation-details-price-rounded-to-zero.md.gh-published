# 022: Offer Operation Details Price Rounded to Zero

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformOperation()` exports offer-operation `details.price` by parsing `xdr.Price.String()`, and the SDK formatter rounds every price to 7 decimal places before the ETL converts it back to `float64`. For valid on-chain prices below `5e-8`, that turns a positive rational into the numeric value `0` while the sibling `details.price_r` field still preserves the true non-zero fraction.

This is silent data corruption in the exported operation details: downstream consumers see a plausible numeric zero rather than a tiny positive price, and the row becomes internally contradictory because `price` and `price_r` disagree.

## Root Cause

`addPriceDetails()` does a lossy string round-trip instead of computing the rational directly from `price.N` and `price.D`. The upstream SDK's `xdr.Price.String()` hard-codes `FloatString(7)`, so values smaller than `0.00000005` become `"0.0000000"` before the ETL calls `strconv.ParseFloat(...)`.

## Reproduction

Any normal export that includes `manage_buy_offer`, `manage_sell_offer`, `create_passive_sell_offer`, or liquidity-pool deposit price bounds with a valid `xdr.Price` below `5e-8` will hit this path. For example, `Price{N:1, D:2147483647}` is a valid positive price (`~4.656612875e-10`), but the exported `details.price` becomes `0` while `details.price_r` still reports `{n:1,d:2147483647}`.

## Affected Code

- `internal/transform/operation.go:addPriceDetails:409-420` — converts `xdr.Price` through `price.String()` and stores the rounded float in `details.price`
- `internal/transform/operation.go:extractOperationDetails:701-749` — offer-operation branches that feed `details.price`
- `internal/transform/operation.go:extractOperationDetails:1008-1011` — liquidity-pool deposit branches that feed `min_price` and `max_price`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/price.go:String:7-10` — upstream formatter rounds rationals to exactly 7 decimal places

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestOfferPriceRoundsToZero`
- **Test language**: `go`
- **How to run**: `cd <repo-root> && go build ./... && go test ./internal/transform/... -run TestOfferPriceRoundsToZero -v` after creating the target test file with the body below.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestOfferPriceRoundsToZero(t *testing.T) {
	tinyPrice := xdr.Price{N: 1, D: 2147483647}
	expected := float64(tinyPrice.N) / float64(tinyPrice.D)
	if expected <= 0 {
		t.Fatalf("expected direct rational division to stay positive, got %e", expected)
	}

	operation := xdr.Operation{
		SourceAccount: &genericSourceAccount,
		Body: xdr.OperationBody{
			Type: xdr.OperationTypeManageSellOffer,
			ManageSellOfferOp: &xdr.ManageSellOfferOp{
				Selling: usdtAsset,
				Buying:  nativeAsset,
				Amount:  1,
				Price:   tinyPrice,
				OfferId: 1,
			},
		},
	}

	envelope := genericBumpOperationEnvelope
	envelope.Tx.Operations = []xdr.Operation{operation}

	transaction := genericLedgerTransaction
	transaction.Envelope.V1 = &envelope

	output, err := TransformOperation(operation, 0, transaction, 0, makeLedgerCloseMeta(), "")
	if err != nil {
		t.Fatalf("TransformOperation returned unexpected error: %v", err)
	}

	priceVal, ok := output.OperationDetails["price"].(float64)
	if !ok {
		t.Fatalf("details[\"price\"] has type %T, want float64", output.OperationDetails["price"])
	}

	priceR, ok := output.OperationDetails["price_r"].(Price)
	if !ok {
		t.Fatalf("details[\"price_r\"] has type %T, want Price", output.OperationDetails["price_r"])
	}

	if output.TypeString != "manage_sell_offer" {
		t.Fatalf("unexpected type_string %q", output.TypeString)
	}

	if priceR.Numerator != int32(tinyPrice.N) || priceR.Denominator != int32(tinyPrice.D) {
		t.Fatalf("price_r = {%d, %d}, want {%d, %d}", priceR.Numerator, priceR.Denominator, tinyPrice.N, tinyPrice.D)
	}

	if priceVal != 0 {
		t.Fatalf("expected rounded export price to be 0, got %e", priceVal)
	}

	t.Logf("bug reproduced: exported price=%v, exact rational=%d/%d, true ratio=%e", priceVal, priceR.Numerator, priceR.Denominator, expected)
}
```

## Expected vs Actual Behavior

- **Expected**: A positive on-chain `xdr.Price` should export as a positive numeric `details.price`, or at minimum not collapse to zero while `details.price_r` remains non-zero.
- **Actual**: Tiny but valid positive prices are exported as `details.price = 0`, while `details.price_r` still contains the true positive rational.

## Adversarial Review

1. Exercises claimed bug: YES — the test drives the production `TransformOperation()` path for a real `manage_sell_offer` operation and inspects the exported details payload.
2. Realistic preconditions: YES — `xdr.Price{N:1, D:2147483647}` is a valid positive rational in the supported on-chain type, and the affected operation types accept arbitrary positive `xdr.Price` values.
3. Bug vs by-design: BUG — Horizon models `price` as a string, but this ETL persists a numeric field and still derives it from a 7-decimal formatter; exporting a positive price as numeric zero is not a defensible numeric contract.
4. Final severity: Critical — this silently writes a materially wrong financial value into exported operation details, and zero can invert downstream analytics or valuation logic.
5. In scope: YES — the ETL emits persisted wrong output without crashing, and the issue is in local transform code rather than an upstream SDK bug.
6. Test correctness: CORRECT — the test uses real XDR types, calls the production transform, verifies the exact rational sibling field, and proves the value is not float-underflow by computing the direct ratio first.
7. Alternative explanations: NONE — the zero comes specifically from the ETL's string round-trip through `FloatString(7)`, not from `float64` inability to represent the ratio.
8. Novelty: NOT ASSESSED HERE — duplicate handling is performed by the orchestrator.

## Suggested Fix

Compute the numeric field directly as `float64(price.N) / float64(price.D)` instead of parsing `price.String()`. If the export intentionally wants Horizon-style formatting, store that rounded value as a string field and avoid presenting it as the authoritative numeric price.
