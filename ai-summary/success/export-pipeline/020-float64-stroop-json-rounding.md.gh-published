# 020: `ConvertStroopValueToReal` rounds large monetary values to the wrong exported JSON number

**Date**: 2026-04-12
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`ConvertStroopValueToReal()` converts exact stroop integers into `float64`, and the export pipeline then serializes those floats with `encoding/json`. For large balances and amounts, that produces numerically wrong JSON output: some values round to the adjacent 7-decimal amount, while larger ones drop the final stroop entirely.

The original hypothesis overstated the exact boundary by using `2^53+1`, which still marshals correctly in some cases. After correcting the PoC to check the actual exported JSON value rather than only an in-memory float round-trip, the data-corruption finding remains real.

## Root Cause

The utility helper `ConvertStroopValueToReal()` returns a `float64` for exact 7-decimal monetary values that originate as integer stroops. The affected export schemas also store those monetary fields as `float64`, so once the transformers call the helper and `ExportEntry()` marshals the result, `encoding/json` can only emit the nearest representable binary float rather than the exact stroop-derived decimal.

## Reproduction

During a normal export, account balances, trustline balances, offer amounts, trade amounts, liquidity-pool reserves, and many operation `details` amount fields all flow through `ConvertStroopValueToReal()`, then through `ExportEntry()`. For sufficiently large values, the serialized JSON number differs from the exact on-chain decimal; for example, `9007199254740997` stroops exports as `900719925.4740998`, and `500000000000000001` stroops exports as `50000000000`.

## Affected Code

- `internal/utils/main.go:84-87` — converts exact stroop integers to `float64`
- `cmd/command_utils.go:55-73` — export path marshals those floats into JSON output
- `internal/transform/schema.go:96-101` — account monetary fields are typed as `float64`
- `internal/transform/account.go:86-90` — account balances and liabilities use the lossy helper
- `internal/transform/trustline.go:70-79` — trustline balances and liabilities use the lossy helper
- `internal/transform/offer.go:84-90` — offer amount uses the lossy helper
- `internal/transform/trade.go:132-145` — trade buy/sell amounts use the lossy helper
- `internal/transform/liquidity_pool.go:66-81` — pool share counts and reserves use the lossy helper
- `internal/transform/operation.go:600-1061` — operation detail amounts reuse the same helper before JSON serialization

## PoC

- **Target test file**: `internal/utils/stroop_precision_poc_test.go`
- **Test name**: `TestConvertStroopValueToReal_LosesPrecisionAbove2e53` and `TestConvertStroopValueToReal_LargeRealisticBalance`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package utils

import (
	"bytes"
	"encoding/json"
	"math/big"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func exportJSONValueRat(t *testing.T, input xdr.Int64) (*big.Rat, string) {
	t.Helper()

	payload := struct {
		Value float64 `json:"value"`
	}{
		Value: ConvertStroopValueToReal(input),
	}

	initialJSON, err := json.Marshal(payload)
	if err != nil {
		t.Fatalf("marshal payload: %v", err)
	}

	decoded := map[string]interface{}{}
	decoder := json.NewDecoder(bytes.NewReader(initialJSON))
	decoder.UseNumber()
	if err := decoder.Decode(&decoded); err != nil {
		t.Fatalf("decode payload: %v", err)
	}

	finalJSON, err := json.Marshal(decoded)
	if err != nil {
		t.Fatalf("re-marshal payload: %v", err)
	}

	var out struct {
		Value json.Number `json:"value"`
	}
	if err := json.Unmarshal(finalJSON, &out); err != nil {
		t.Fatalf("unmarshal final payload: %v", err)
	}

	got, ok := new(big.Rat).SetString(out.Value.String())
	if !ok {
		t.Fatalf("parse exported number %q", out.Value.String())
	}

	return got, string(finalJSON)
}

func exactValueRat(input xdr.Int64) *big.Rat {
	return new(big.Rat).SetFrac64(int64(input), 10_000_000)
}

func stroopDelta(exact, got *big.Rat) *big.Rat {
	delta := new(big.Rat).Sub(exact, got)
	delta.Abs(delta)
	return delta.Mul(delta, big.NewRat(10_000_000, 1))
}

func TestConvertStroopValueToReal_LosesPrecisionAbove2e53(t *testing.T) {
	const inputStroops xdr.Int64 = 9_007_199_254_740_997 // 2^53 + 5

	exact := exactValueRat(inputStroops)
	got, exportedJSON := exportJSONValueRat(t, inputStroops)

	if exact.Cmp(got) == 0 {
		t.Fatalf("expected exported JSON to lose precision for %d stroops, but it stayed exact: %s", inputStroops, exportedJSON)
	}

	if got.FloatString(7) != "900719925.4740998" {
		t.Fatalf("expected rounded JSON value 900719925.4740998, got %s (json=%s)", got.FloatString(7), exportedJSON)
	}

	if delta := stroopDelta(exact, got); delta.Cmp(big.NewRat(1, 1)) != 0 {
		t.Fatalf("expected 1 stroop of error, got %s stroops (json=%s)", delta.FloatString(0), exportedJSON)
	}
}

func TestConvertStroopValueToReal_LargeRealisticBalance(t *testing.T) {
	const inputStroops xdr.Int64 = 500_000_000_000_000_001 // 50B XLM + 1 stroop

	exact := exactValueRat(inputStroops)
	got, exportedJSON := exportJSONValueRat(t, inputStroops)

	if exact.Cmp(got) == 0 {
		t.Fatalf("expected exported JSON to lose precision for %d stroops, but it stayed exact: %s", inputStroops, exportedJSON)
	}

	if got.FloatString(7) != "50000000000.0000000" {
		t.Fatalf("expected exported JSON to drop the fractional stroop, got %s (json=%s)", got.FloatString(7), exportedJSON)
	}

	if delta := stroopDelta(exact, got); delta.Cmp(big.NewRat(1, 1)) != 0 {
		t.Fatalf("expected 1 stroop of error, got %s stroops (json=%s)", delta.FloatString(0), exportedJSON)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Monetary exports should preserve the exact 7-decimal value implied by the on-chain stroop integer.
- **Actual**: Large monetary fields are serialized from `float64`, so the JSON output rounds to a nearby decimal and can differ from the exact on-chain value by one or more stroops.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses the production conversion helper and the same `json.Marshal`/`UseNumber` export sequence used by `ExportEntry()`, then compares the exported JSON number against the exact rational value.
2. Realistic preconditions: YES — balances and amounts above roughly hundreds of millions of XLM are plausible for major treasury accounts, issuer trustlines, pool reserves, and large custom assets; `50B XLM + 1 stroop` is a valid on-chain amount.
3. Bug vs by-design: BUG — the exporter silently emits incorrect monetary values with no lossless companion field for the affected datasets, so downstream consumers cannot recover the original stroop amount from the exported row.
4. Final severity: Critical — this is direct corruption of monetary fields (`balance`, `amount`, `selling_amount`, `buying_amount`, pool reserves, liabilities) in normal exports.
5. In scope: YES — it is a concrete production export path that yields wrong but plausible output.
6. Test correctness: CORRECT — the revised PoC avoids the original false lead (`2^53+1` can still serialize correctly) and asserts on the exported JSON number itself, not merely on binary float round-trips or formatting differences.
7. Alternative explanations: NONE — the test compares exact rational values, so it excludes harmless formatting elision such as dropping trailing `.0000000`.
8. Novelty: NOVEL

## Suggested Fix

Stop exporting monetary values as `float64`. Preserve the exact stroop integer (or emit a decimal/string field derived without binary floating point), and update the affected schemas and transformers so account, trustline, offer, trade, liquidity-pool, and operation amount fields remain lossless end-to-end.
