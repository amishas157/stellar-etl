# 009: SEP-41 custom token amounts scaled as stroops

**Date**: 2026-04-10
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`transformEvents()` converts every token-transfer `AmountRaw` string into `TokenTransferOutput.Amount` by multiplying it by `1e-7`. That is correct for SAC/native token events whose SDK amount strings are stroop-denominated, but it silently corrupts non-SAC SEP-41 events because the SDK preserves their contract-emitted `i128` amount as a raw decimal string such as `"1000"`.

As a result, a custom token transfer that should export `amount = 1000` is emitted as `0.0001`, while `amount_raw` still shows `"1000"`. The row looks plausible and does not raise an error, but the primary numeric field is wrong by a factor of 10,000,000.

## Root Cause

`transformEvents()` applies the same Stellar stroop conversion to every token event before it even checks whether the event has a validated Stellar asset attached. The code already distinguishes custom SEP-41 events from SAC/native ones via `event.GetAsset() == nil` / empty asset fields, but the amount conversion path ignores that distinction and always divides by `10^7`.

## Reproduction

During normal operation, this manifests whenever `export_token_transfer` processes a non-SAC SEP-41 transfer/mint/burn/clawback event. The upstream token-transfer processor emits such events with `event.GetAsset() == nil` and an unscaled raw amount string (for example `"1000"`); the ETL then exports `amount = 0.0001` instead of `1000`.

## Affected Code

- `internal/transform/token_transfer.go:transformEvents:47-73` — parses each event amount string and unconditionally multiplies it by `0.0000001`
- `internal/transform/token_transfer.go:transformEvents:108-126` — writes the mis-scaled float into `TokenTransferOutput.Amount`
- `internal/transform/token_transfer.go:getAssetFromEvent:132-150` — already exposes the custom-token/SAC distinction, but the amount conversion path does not use it

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestCustomSEP41TokenAmountScaledAsStroops`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./internal/transform/... -run TestCustomSEP41TokenAmountScaledAsStroops -v`
  4. Observe `AmountRaw` remains `"1000"` while `Amount` is exported as `9.999999999999999e-05` instead of `1000`.

### Test Body

```go
package transform

import (
	"strconv"
	"testing"
	"time"

	"github.com/guregu/null"
	"github.com/stellar/go-stellar-sdk/processors/token_transfer"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestCustomSEP41TokenAmountScaledAsStroops demonstrates that transformEvents
// unconditionally multiplies all token amounts by 1e-7 (stroop scaling), even
// for custom SEP-41 tokens whose AmountRaw is already in raw token units.
// A custom token transfer with Amount="1000" should yield Amount=1000.0, but
// the code produces Amount=0.0001.
func TestCustomSEP41TokenAmountScaledAsStroops(t *testing.T) {
	operationIndex := uint32(1)

	// Custom SEP-41 token event: Asset is nil (no SAC asset)
	customTokenEvents := []*token_transfer.TokenTransferEvent{
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash",
				TransactionIndex: 1,
				OperationIndex:   &operationIndex,
				ContractAddress:  "customcontract",
			},
			Event: &token_transfer.TokenTransferEvent_Transfer{
				Transfer: &token_transfer.Transfer{
					From:   "from",
					To:     "to",
					Asset:  nil, // custom SEP-41: no SAC asset
					Amount: "1000",
				},
			},
		},
	}

	lcm := xdr.LedgerCloseMeta{
		V: 1,
		V1: &xdr.LedgerCloseMetaV1{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue: xdr.StellarValue{
						CloseTime: 1000,
					},
					LedgerSeq: 10,
				},
			},
		},
	}

	outputs, err := transformEvents(customTokenEvents, lcm)
	if err != nil {
		t.Fatalf("transformEvents returned error: %v", err)
	}

	if len(outputs) != 1 {
		t.Fatalf("expected 1 output, got %d", len(outputs))
	}

	output := outputs[0]

	// Verify AmountRaw is preserved correctly
	if output.AmountRaw != "1000" {
		t.Errorf("AmountRaw corrupted: got %q, want %q", output.AmountRaw, "1000")
	}

	// Verify that AssetType is empty (custom token, not SAC)
	if output.AssetType != "" {
		t.Errorf("AssetType should be empty for custom SEP-41 token, got %q", output.AssetType)
	}

	// BUG DEMONSTRATION: Amount is incorrectly scaled by 1e-7.
	// For a custom SEP-41 token, Amount should be 1000.0 (raw token units),
	// but transformEvents unconditionally applies stroop scaling.
	// Reproduce the exact runtime float64: ParseFloat then multiply
	expectedBuggyFloat, _ := strconv.ParseFloat("1000", 64)
	expectedBuggyFloat = expectedBuggyFloat * 0.0000001
	correctAmount := 1000.0 // What it SHOULD be

	if output.Amount != expectedBuggyFloat {
		t.Errorf("Expected buggy Amount=%v (stroop-scaled), got %v", expectedBuggyFloat, output.Amount)
	}

	if output.Amount == correctAmount {
		t.Error("Amount equals correct value 1000.0 — bug may have been fixed")
	}

	// Also verify the expected output structure for completeness
	expectedOutput := TokenTransferOutput{
		TransactionHash: "txhash",
		TransactionID:   42949677056,
		OperationID:     null.IntFrom(42949677057),
		EventTopic:      "transfer",
		From:            null.StringFrom("from"),
		To:              null.StringFrom("to"),
		Asset:           "",
		AssetType:       "",
		AssetCode:       null.NewString("", false),
		AssetIssuer:     null.NewString("", false),
		Amount:          expectedBuggyFloat,
		AmountRaw:       "1000",
		ContractID:      "customcontract",
		LedgerSequence:  uint32(10),
		ClosedAt:        time.Unix(1000, 0).UTC(),
		ToMuxed:         null.NewString("", false),
		ToMuxedID:       null.NewString("", false),
	}

	if output != expectedOutput {
		t.Errorf("Output mismatch.\nGot:  %+v\nWant: %+v", output, expectedOutput)
	}

	t.Logf("BUG CONFIRMED: Custom SEP-41 token with AmountRaw=%q produces Amount=%v (should be %v)",
		output.AmountRaw, output.Amount, correctAmount)
	t.Logf("The amount is incorrectly divided by 10^7 because transformEvents applies stroop scaling unconditionally")
}
```

## Expected vs Actual Behavior

- **Expected**: Non-SAC SEP-41 events preserve the custom token quantity represented by the SDK event amount string, so a raw `"1000"` exports as `amount = 1000`.
- **Actual**: The exporter always applies Stellar stroop scaling and emits `amount = 9.999999999999999e-05` while `amount_raw = "1000"`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the exact production transformer that populates `TokenTransferOutput.Amount` and demonstrates the unconditional `1e-7` scaling on a custom-token event.
2. Realistic preconditions: YES — the upstream token-transfer processor's own tests show non-SAC SEP-41 V3/V4 events are emitted with `event.GetAsset() == nil` and raw amount strings like `"1000"`, so this input shape occurs in normal exports.
3. Bug vs by-design: BUG — the repository already exports custom token events (empty asset fields plus populated `contract_id`), and there is no code-level contract justifying conversion of arbitrary custom-token integers into Stellar stroops.
4. Final severity: Critical — this silently corrupts a monetary amount field in exported token-transfer rows.
5. In scope: YES — it is a concrete export-pipeline data-correctness defect with a real production code path.
6. Test correctness: CORRECT — the test does not assert a tautology; it proves `AmountRaw` and `Amount` diverge exactly at the transformer’s scaling step.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Only apply the `1e-7` stroop conversion for events backed by a validated Stellar asset (native/SAC). For non-SAC SEP-41 events where `event.GetAsset() == nil`, preserve the raw parsed amount instead of dividing by `10^7`, or export an explicit token-decimals-aware field if a different normalization is desired.
