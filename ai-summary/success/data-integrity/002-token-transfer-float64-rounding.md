# 002: Token transfer float64 rounding exports the wrong amount

**Date**: 2026-04-10
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`transformEvents()` converts token-transfer amounts by calling `strconv.ParseFloat(amount, 64)` on the raw decimal string and then multiplying by `1e-7`. For large values this introduces an avoidable double-rounding step before export, so the emitted `amount` JSON number can differ from the exact decimal implied by `amount_raw`.

This is observable in the normal export path because `ExportEntry()` serializes the resulting `float64` with `json.Marshal`. For `amount_raw = "9007199254740993"`, the exported `amount` becomes `900719925.4740992` even though the single-rounded decimal value is `900719925.4740993`.

## Root Cause

`internal/transform/token_transfer.go` parses the string amount directly to `float64`, which rounds large integers at the parse step once they exceed `float64`'s exact integer range. It then scales that already-rounded value by `1e-7` and stores it in `TokenTransferOutput.Amount`, so the exported numeric field can diverge from the exact raw amount preserved in `AmountRaw`.

## Reproduction

During normal token-transfer export, any transfer, mint, burn, clawback, or fee event with a sufficiently large raw amount string will hit this path. When the raw amount exceeds the exact integer range of `float64` (for example `9007199254740993` stroops), the exported `amount` field is serialized as a different decimal value than `amount_raw / 10_000_000`.

## Affected Code

- `internal/transform/token_transfer.go:transformEvents:37-126` — parses `Amount` through `strconv.ParseFloat` and stores the scaled `float64` in `TokenTransferOutput.Amount`
- `internal/transform/schema.go:TokenTransferOutput:659-677` — exposes the lossy `amount` field beside the exact `amount_raw` string
- `cmd/command_utils.go:ExportEntry:55-80` — marshals the transformed struct to JSON, making the rounded `amount` value user-visible in exports

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTokenTransferFloat64PrecisionLoss`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"encoding/json"
	"math/big"
	"strconv"
	"testing"
)

func TestTokenTransferFloat64PrecisionLoss(t *testing.T) {
	tests := []struct {
		name      string
		rawAmount string
		wantJSON  string
	}{
		{
			name:      "2^53_plus_1",
			rawAmount: "9007199254740993",
			wantJSON:  `{"amount":900719925.4740992}`,
		},
		{
			name:      "2^54_plus_1",
			rawAmount: "18014398509481985",
			wantJSON:  `{"amount":1801439850.9481983}`,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			events, ledgers, err := makeTokenTransferTestInput()
			if err != nil {
				t.Fatalf("makeTokenTransferTestInput failed: %v", err)
			}

			events[0][0].GetTransfer().Amount = tt.rawAmount

			outputs, err := transformEvents(events[0][:1], ledgers[0])
			if err != nil {
				t.Fatalf("transformEvents failed: %v", err)
			}
			if len(outputs) != 1 {
				t.Fatalf("expected 1 output, got %d", len(outputs))
			}

			out := outputs[0]
			if out.AmountRaw != tt.rawAmount {
				t.Fatalf("AmountRaw mismatch: got %q, want %q", out.AmountRaw, tt.rawAmount)
			}

			productionFloat, err := strconv.ParseFloat(tt.rawAmount, 64)
			if err != nil {
				t.Fatalf("ParseFloat failed: %v", err)
			}
			productionFloat *= 1e-7
			if out.Amount != productionFloat {
				t.Fatalf("Amount mismatch: got %.20f, want %.20f", out.Amount, productionFloat)
			}

			rawInt, ok := new(big.Int).SetString(tt.rawAmount, 10)
			if !ok {
				t.Fatalf("failed to parse %q as big.Int", tt.rawAmount)
			}
			exactFloat, _ := new(big.Rat).SetFrac(rawInt, big.NewInt(10_000_000)).Float64()
			if out.Amount == exactFloat {
				t.Fatalf("expected double-rounded output to differ from single-rounding float64 for %s", tt.rawAmount)
			}

			original, err := strconv.ParseInt(tt.rawAmount, 10, 64)
			if err != nil {
				t.Fatalf("ParseInt failed: %v", err)
			}
			if got := int64(out.Amount * 1e7); got == original {
				t.Fatalf("expected round-trip stroops to differ, got %d", got)
			}

			payload, err := json.Marshal(struct {
				Amount float64 `json:"amount"`
			}{Amount: out.Amount})
			if err != nil {
				t.Fatalf("Marshal failed: %v", err)
			}
			if string(payload) != tt.wantJSON {
				t.Fatalf("unexpected JSON payload: got %s, want %s", payload, tt.wantJSON)
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: The exported `amount` field should represent the decimal value implied by `amount_raw / 10_000_000`, so `9007199254740993` should export as `900719925.4740993`.
- **Actual**: The current conversion path exports `900719925.4740992` for that raw amount because it rounds once during `ParseFloat` and again during scaling.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the production `transformEvents()` path and then serializes the resulting `amount` exactly as the exporter does.
2. Realistic preconditions: YES — token-transfer events arrive with `Amount` as a string, and large issued-asset or Soroban token values can exceed `float64`'s exact integer range during normal exports.
3. Bug vs by-design: BUG — the schema may choose to expose a `float64`, but this implementation adds avoidable extra precision loss beyond that design by parsing to `float64` before scaling instead of converting once from exact integer data.
4. Final severity: Critical — the exported monetary `amount` field contains a wrong numeric value for real inputs, which can mislead downstream analytics that read the numeric field.
5. In scope: YES — this is silent financial data corruption in exported output.
6. Test correctness: CORRECT — the test uses a real token-transfer event, invokes production code, checks the emitted JSON number, and verifies the mismatch is not tautological.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

At minimum, parse the raw amount into exact integer/rational form first and convert to `float64` only once, e.g. via `big.Int`/`big.Rat`, and return parse errors instead of discarding them. If exact decimal fidelity is required for all token amounts, the long-term fix is to export token amounts as a decimal string or integer-based field rather than `float64`.
