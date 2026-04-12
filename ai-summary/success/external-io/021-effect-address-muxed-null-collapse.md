# 021: Effect Parquet rewrites missing `address_muxed` to empty strings

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_effects --write-parquet` preserves muxed-address presence in the transform layer with `null.String`, but the Parquet conversion rewrites every unmuxed effect row to `address_muxed=""`. That silently changes missing data into a concrete string value while still emitting real `M...` addresses for muxed rows.

This is structural data corruption in normal export flow. Most effect rows come from classic `G...` accounts, so the broken Parquet value appears broadly and looks plausible to downstream readers.

## Root Cause

`effectsWrapper.addUnmuxed()` intentionally stores `null.String{}` for classic-account effects, while `addMuxed()` stores a real muxed address only when one exists. `EffectOutputParquet` narrows that nullable field to a required `string`, and `EffectOutput.ToParquet()` writes `eo.AddressMuxed.String`, collapsing `Valid=false` into `""`.

## Reproduction

During normal operation, any effect row produced for a classic `G...` account takes the `addUnmuxed()` path and carries an invalid `null.String` in `EffectOutput.AddressMuxed`. The JSON/export schema preserves that missing/null state, but the Parquet export converts the same row into `address_muxed=""`.

## Affected Code

- `internal/transform/effects.go:addUnmuxed/addMuxed:187-197` — classic-account effects keep `AddressMuxed` invalid while muxed-account effects populate it.
- `internal/transform/schema.go:EffectOutput:361-371` — output schema models `address_muxed` as `null.String`.
- `internal/transform/parquet_converter.go:EffectOutput.ToParquet:277-289` — converter serializes `eo.AddressMuxed.String`, flattening invalid values to `""`.
- `internal/transform/schema_parquet.go:EffectOutputParquet:247-257` — Parquet schema defines `address_muxed` as a required `string`.
- `cmd/export_effects.go:33-68` — production export path accumulates effect outputs and writes `EffectOutputParquet` rows.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestEffectAddressMuxedNullCollapse`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestEffectAddressMuxedNullCollapse -v`.

### Test Body

```go
package transform

import (
	"bytes"
	"encoding/json"
	"testing"

	"github.com/guregu/null"
)

func TestEffectAddressMuxedNullCollapse(t *testing.T) {
	unmuxed := EffectOutput{
		Address:      "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
		AddressMuxed: null.String{},
		OperationID:  12345,
		Type:         1,
		TypeString:   "account_created",
	}

	muxedAddress := "MAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCAAAAAAAAAPCICBKU"
	muxed := unmuxed
	muxed.OperationID = 12346
	muxed.AddressMuxed = null.StringFrom(muxedAddress)

	unmuxedJSON, err := json.Marshal(unmuxed)
	if err != nil {
		t.Fatalf("marshal unmuxed effect: %v", err)
	}
	muxedJSON, err := json.Marshal(muxed)
	if err != nil {
		t.Fatalf("marshal muxed effect: %v", err)
	}

	if bytes.Contains(unmuxedJSON, []byte(`"address_muxed":""`)) {
		t.Fatalf("json already flattened address_muxed to empty string: %s", string(unmuxedJSON))
	}

	var unmuxedRow map[string]any
	if err := json.Unmarshal(unmuxedJSON, &unmuxedRow); err != nil {
		t.Fatalf("unmarshal unmuxed json: %v", err)
	}
	if value, ok := unmuxedRow["address_muxed"]; ok && value != nil {
		t.Fatalf("expected unmuxed json to preserve missing/null address_muxed, got %#v in %s", value, string(unmuxedJSON))
	}

	var muxedRow map[string]any
	if err := json.Unmarshal(muxedJSON, &muxedRow); err != nil {
		t.Fatalf("unmarshal muxed json: %v", err)
	}
	if muxedRow["address_muxed"] != muxedAddress {
		t.Fatalf("expected muxed json to preserve address_muxed=%q, got %#v in %s", muxedAddress, muxedRow["address_muxed"], string(muxedJSON))
	}

	unmuxedParquet := unmuxed.ToParquet().(EffectOutputParquet)
	muxedParquet := muxed.ToParquet().(EffectOutputParquet)

	if unmuxedParquet.AddressMuxed != "" {
		t.Fatalf("expected unmuxed parquet address_muxed to flatten to empty string, got %q", unmuxedParquet.AddressMuxed)
	}
	if muxedParquet.AddressMuxed != muxedAddress {
		t.Fatalf("expected muxed parquet address_muxed=%q, got %q", muxedAddress, muxedParquet.AddressMuxed)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Parquet should preserve the missing/null `address_muxed` state for classic-account effect rows, just as the transform/JSON layer does.
- **Actual**: Parquet rewrites classic-account rows to `address_muxed=""` while muxed-account rows keep real `M...` addresses.

## Adversarial Review

1. Exercises claimed bug: YES — the test hits `EffectOutput.ToParquet()`, the production conversion code that flattens `null.String`, and uses the exact field states produced by `addUnmuxed()` and `addMuxed()`.
2. Realistic preconditions: YES — classic-account effects are the common case, and `effects.go` explicitly emits `null.String{}` for them during normal effect generation.
3. Bug vs by-design: BUG — the transform schema models `address_muxed` as nullable, and there is no repository documentation defining `""` as the intended Parquet sentinel.
4. Final severity: High — this is silent structural corruption of a non-financial attribution field in exported data.
5. In scope: YES — it is a concrete exporter correctness bug that produces plausible but wrong Parquet rows.
6. Test correctness: CORRECT — the PoC compares the same production value before and after Parquet conversion, so it does not pass because of circular setup or a tautological assertion.
7. Alternative explanations: NONE — the observed value change comes directly from `.AddressMuxed.String` plus a non-optional Parquet `string` column.
8. Novelty: NOT ASSESSED HERE — duplicate handling is delegated to the orchestrator.

## Suggested Fix

Make `EffectOutputParquet.AddressMuxed` nullable, such as `*string` with an optional Parquet tag, and emit `nil` when `AddressMuxed.Valid` is false. If nullable Parquet columns are not viable, add a parallel validity column so downstream consumers can distinguish absent values from concrete strings.
