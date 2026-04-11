# H001: Token-transfer JSON `amount` rounds large raw i128 values through `float64`

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every token-transfer row, `amount` should be the exact decimal rendering of `amount_raw` scaled by 1e7, so two distinct on-chain raw amounts always export as two distinct numeric amounts.

## Mechanism

`transformEvents()` converts the raw i128 decimal string into a `float64` with `strconv.ParseFloat`, then multiplies by `0.0000001` before storing it in `TokenTransferOutput.Amount`. Large integers above the 53-bit mantissa limit cannot be represented exactly, so low-order digits are rounded away even though `amount_raw` still carries the exact source value; downstream consumers who read `amount` get a plausible but wrong monetary value.

## Trigger

Export any token transfer whose raw event amount needs more than 53 bits of integer precision, such as `amount_raw = "9007199254740993"` or `amount_raw = "9223372036854775807"`. In Go, those parse as `9007199254740992` and `9223372036854775808` respectively before the 1e7 scale factor is applied.

## Target Code

- `internal/transform/token_transfer.go:44-73` — parses raw event amounts with `strconv.ParseFloat` and rescales them into `float64`
- `internal/transform/token_transfer.go:108-126` — exports the rounded float into `TokenTransferOutput.Amount` while also preserving the exact `AmountRaw`
- `internal/transform/schema.go:659-677` — publishes `amount` as a numeric JSON field that downstream systems can consume directly

## Evidence

The export path keeps the exact source string in `AmountRaw` but still derives `Amount` through a lossy `float64` conversion. A direct Go reproduction shows `strconv.ParseFloat("9007199254740993", 64)` becomes `9007199254740992`, so the exported scaled amount is rounded before JSON encoding ever happens.

## Anti-Evidence

`AmountRaw` remains exact, so consumers that deliberately ignore `amount` can recover the source integer. The bug matters specifically because the ETL also emits a first-class numeric `amount` field that looks authoritative and is easier for analytics pipelines to use.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete token transfer export path from the upstream SDK's `amount.String128Raw()` through `transformEvents()` in `token_transfer.go` to the final `TokenTransferOutput` struct. The SDK extracts i128 values from Soroban contract events and emits them as raw decimal strings (confirmed at `go-stellar-sdk/processors/token_transfer/contract_events.go:82`). The ETL then parses these arbitrary-precision integer strings through `strconv.ParseFloat(amount, 64)` on lines 52/57/62/67/72, losing precision for any value exceeding 2^53 (≈9.007×10^15). The lossy float64 is multiplied by 0.0000001 and stored in `Amount`, while the exact string is preserved in `AmountRaw`.

### Code Paths Examined

- `go-stellar-sdk/processors/token_transfer/contract_events.go:78-82` — Upstream SDK extracts i128 from XDR via `value.GetI128()` and converts to decimal string via `amount.String128Raw(amt)`. This string can represent values up to 2^127-1.
- `internal/transform/token_transfer.go:37-76` — `transformEvents()` switches on event type (Transfer/Mint/Burn/Clawback/Fee). Each arm reads `evt.*.Amount` (a raw i128 decimal string), then calls `strconv.ParseFloat(amount, 64)` discarding the error, and multiplies by 0.0000001. No range check or precision guard exists.
- `internal/transform/token_transfer.go:108-126` — The lossy `amountFloat` is assigned to `TokenTransferOutput.Amount` (float64) while the exact string goes to `AmountRaw`.
- `internal/transform/schema.go:659-677` — `TokenTransferOutput.Amount` is typed `float64` with json tag `"amount"`, exported directly to JSON consumers.
- No Parquet schema exists for `TokenTransferOutput` (grep confirmed no match in `schema_parquet.go`), so this affects the JSON export path only.

### Findings

The bug is confirmed:

1. **Lossy conversion**: `strconv.ParseFloat` converts an arbitrary-precision integer string to float64, which has only 53 bits of mantissa. Any raw amount ≥ 2^53 = 9,007,199,254,740,993 stroops (~900.7 million XLM) loses precision. For Soroban custom tokens with different decimal semantics, the raw i128 value can easily exceed this threshold.

2. **Silent error discarding**: The error return from `strconv.ParseFloat` is discarded with `_` (lines 52, 57, 62, 67, 72). For values that overflow float64 range entirely (theoretically possible with i128), this would silently produce `+Inf` or `0` with no error propagation.

3. **Authoritative field exposure**: The `Amount` field is a first-class JSON member that downstream analytics systems (BigQuery, compliance pipelines) will naturally prefer over the string `AmountRaw` for numeric operations. The field looks authoritative but silently carries rounded values.

4. **Scope of impact**: For classic XLM (7-digit precision), the threshold of ~900 million XLM is high but achievable (SDF manages billions of XLM). For Soroban tokens with non-7-digit decimal schemes, the raw i128 values routinely exceed 2^53, making this a common rather than edge-case bug.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go`
- **Setup**: Construct a `token_transfer.TokenTransferEvent` with a Transfer event whose `Amount` string is `"9007199254740993"` (2^53 + 1). Build minimal `LedgerCloseMeta` with a valid ledger sequence and closed-at time.
- **Steps**: Call `transformEvents()` with the crafted event. Extract the `Amount` and `AmountRaw` fields from the returned `TokenTransferOutput`.
- **Assertion**: Assert that `output.AmountRaw == "9007199254740993"` (exact) but `output.Amount != 900719925.4740993` (the mathematically correct scaled value). Specifically, `output.Amount` will equal `900719925.4740992` due to float64 rounding. This demonstrates that two distinct on-chain amounts (`9007199254740992` and `9007199254740993`) produce identical `Amount` values in the export.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/token_transfer_float64_poc_test.go
**Test Name**: "TestTokenTransferAmountFloat64Rounding"
**Test Language**: Go

### Demonstration

The test constructs a token transfer event with `Amount = "9007199254740993"` (2^53 + 1), passes it through the production `transformEvents()` code path, and confirms that the output `Amount` field (900719925.4740992) differs from the mathematically correct scaled value (900719925.4740993). It further proves that two distinct raw amounts ("9007199254740992" and "9007199254740993") collide to the same float64 `Amount`, meaning downstream consumers cannot distinguish them.

### Test Body

```go
package transform

import (
	"strconv"
	"testing"
	"time"

	"github.com/guregu/null"
	"github.com/stellar/go-stellar-sdk/asset"
	"github.com/stellar/go-stellar-sdk/processors/token_transfer"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTokenTransferAmountFloat64Rounding demonstrates that large raw i128
// amounts lose precision when parsed through strconv.ParseFloat and scaled by
// 1e-7, producing a rounded Amount that does not match the mathematically
// correct value derived from AmountRaw.
func TestTokenTransferAmountFloat64Rounding(t *testing.T) {
	// 2^53 + 1 — the smallest positive integer that cannot be represented
	// exactly in a float64 mantissa. strconv.ParseFloat rounds it to 2^53.
	const rawAmount = "9007199254740993"

	// Mathematically correct scaled value: 9007199254740993 * 1e-7 = 900719925.4740993
	// But float64 rounds the parsed integer to 9007199254740992, so the scaled
	// value becomes 900719925.4740992.
	correctScaled := 900719925.4740993

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
					Amount: rawAmount,
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

	outputs, err := transformEvents(events, lcm)
	if err != nil {
		t.Fatalf("transformEvents returned error: %v", err)
	}
	if len(outputs) != 1 {
		t.Fatalf("expected 1 output, got %d", len(outputs))
	}

	out := outputs[0]

	// AmountRaw should preserve the exact source string.
	if out.AmountRaw != rawAmount {
		t.Errorf("AmountRaw corrupted: got %q, want %q", out.AmountRaw, rawAmount)
	}

	// The bug: Amount should be the mathematically correct scaled value
	// (900719925.4740993) but instead equals a rounded value (900719925.4740992)
	// due to float64 precision loss in strconv.ParseFloat.
	if out.Amount == correctScaled {
		t.Errorf("Amount unexpectedly matches the correct value %v; expected float64 rounding", correctScaled)
	} else {
		t.Logf("BUG CONFIRMED: Amount = %.10f, correct = %.10f (difference = %e)",
			out.Amount, correctScaled, correctScaled-out.Amount)
	}

	// Additionally demonstrate that two distinct raw amounts map to the same
	// Amount, proving data corruption for downstream consumers.
	rawAmountAdjacent := "9007199254740992" // 2^53 — one less than rawAmount
	adjacentFloat, _ := strconv.ParseFloat(rawAmountAdjacent, 64)
	adjacentScaled := adjacentFloat * 0.0000001

	if out.Amount == adjacentScaled {
		t.Logf("COLLISION CONFIRMED: raw %s and raw %s both produce Amount = %.10f",
			rawAmount, rawAmountAdjacent, out.Amount)
	}

	// Verify expected output for completeness
	expected := TokenTransferOutput{
		TransactionHash: "txhash",
		TransactionID:   42949677056,
		OperationID:     null.IntFrom(42949677057),
		EventTopic:      "transfer",
		From:            null.StringFrom("from"),
		To:              null.StringFrom("to"),
		Asset:           "credit_alphanum4:abc:def",
		AssetType:       "credit_alphanum4",
		AssetCode:       null.StringFrom("abc"),
		AssetIssuer:     null.StringFrom("def"),
		Amount:          900719925.4740992, // rounded, NOT the correct 900719925.4740993
		AmountRaw:       rawAmount,
		ContractID:      "contractaddress",
		LedgerSequence:  uint32(10),
		ClosedAt:        time.Unix(1000, 0).UTC(),
		ToMuxed:         null.NewString("", false),
		ToMuxedID:       null.NewString("", false),
	}

	if out != expected {
		t.Errorf("output mismatch:\ngot  %+v\nwant %+v", out, expected)
	}
}
```

### Test Output

```
=== RUN   TestTokenTransferAmountFloat64Rounding
    token_transfer_float64_poc_test.go:91: BUG CONFIRMED: Amount = 900719925.4740991592, correct = 900719925.4740992785 (difference = 1.192093e-07)
    token_transfer_float64_poc_test.go:102: COLLISION CONFIRMED: raw 9007199254740993 and raw 9007199254740992 both produce Amount = 900719925.4740991592
--- PASS: TestTokenTransferAmountFloat64Rounding (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.704s
```
