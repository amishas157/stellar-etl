# H020: Effect Parquet rewrites missing `address_muxed` to an empty string

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Effects created for ordinary `G...` accounts should preserve the fact that no muxed address exists. The Parquet row should distinguish "no muxed address" from a real `M...` address, matching the JSON export semantics.

## Mechanism

The effects pipeline stores muxed-address metadata as `null.String` and explicitly passes `null.String{}` on unmuxed paths. The Parquet schema instead uses a plain `string`, and `EffectOutput.ToParquet()` serializes `AddressMuxed.String`, which turns every missing value into `""`. That means a field that is absent in the source row is exported as a concrete string value in Parquet.

## Trigger

Run `export_effects --write-parquet` on any ledger range containing normal account effects without muxed participants. The JSON output will preserve the field as missing/null, while the Parquet output will emit `address_muxed=""` for those same rows.

## Target Code

- `internal/transform/effects.go:effectsWrapper.add:176-184` — stores muxed-address metadata in a nullable wrapper
- `internal/transform/effects.go:effectsWrapper.addUnmuxed:187-189` — emits `null.String{}` for non-muxed addresses
- `internal/transform/schema.go:EffectOutput:361-371` — models `address_muxed` as `null.String`
- `internal/transform/schema_parquet.go:EffectOutputParquet:247-257` — defines `address_muxed` as a plain `string`
- `internal/transform/parquet_converter.go:EffectOutput.ToParquet:277-289` — flattens the nullable wrapper via `.String`

## Evidence

The effect transform uses separate `addUnmuxed` and `addMuxed` helpers, which only makes sense if the export contract is supposed to preserve whether a muxed address exists. The JSON schema keeps `AddressMuxed` nullable, while the Parquet converter always writes a string value. Existing effect tests include populated muxed-address cases, showing that the field is semantically meaningful when present.

## Anti-Evidence

An empty string is not a valid muxed address, so some consumers may be able to treat `""` as a de facto sentinel. But the exporter never documents that sentinel, and Parquet no longer faithfully represents the original missing state.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated for effect `address_muxed` (same bug class as confirmed findings 013-trade-parquet-null-collapse and 014-sponsor-null-collapses-to-empty-string, but distinct field)

### Trace Summary

Traced the complete code path from effect creation through Parquet serialization. `effectsWrapper.addUnmuxed()` at effects.go:188 passes `null.String{}` (Valid=false, String="") for non-muxed accounts. This flows into `EffectOutput.AddressMuxed` (type `null.String` at schema.go:363). When `ToParquet()` at parquet_converter.go:280 reads `eo.AddressMuxed.String`, it extracts the embedded empty string. The Parquet schema at schema_parquet.go:249 defines `AddressMuxed` as a plain `string` with no nullability support, so the empty string is written as a concrete value. JSON serialization via `null.String`'s custom marshaler correctly emits `null` for the same row.

### Code Paths Examined

- `internal/transform/effects.go:addUnmuxed:187-189` — passes `null.String{}` (Valid=false) for unmuxed accounts
- `internal/transform/effects.go:addMuxed:191-198` — passes `null.StringFrom(address.Address())` (Valid=true) for muxed accounts; confirms the two helpers intentionally preserve the null/valid distinction
- `internal/transform/effects.go:add:176-184` — stores `addressMuxed` in `EffectOutput.AddressMuxed` without modification
- `internal/transform/schema.go:EffectOutput:363` — `AddressMuxed null.String` with `json:"address_muxed,omitempty"` — JSON serialization correctly emits null when Valid=false
- `internal/transform/schema_parquet.go:EffectOutputParquet:249` — `AddressMuxed string` — plain string, no nullable Parquet annotation
- `internal/transform/parquet_converter.go:ToParquet:280` — `AddressMuxed: eo.AddressMuxed.String` — reads the embedded `String` field directly, which is `""` when `Valid` is false

### Findings

1. **Null collapse confirmed**: When `addUnmuxed()` is called (the common path for all non-muxed accounts), `AddressMuxed` is `null.String{Valid: false, String: ""}`. JSON serialization correctly preserves this as `null` (omitted via `omitempty`). Parquet serialization reads `.String` and emits `""`.

2. **Vast majority of effects affected**: Most Stellar effects involve non-muxed `G...` accounts. Only effects for `M...` muxed accounts populate `AddressMuxed`. This means the overwhelming majority of Parquet effect rows will carry `address_muxed=""` instead of a null marker.

3. **Same root cause as confirmed findings**: This is the identical pattern confirmed in success findings 013 (trade null collapse), 014 (sponsor null collapse), and 019 (transaction min_account_sequence). The Parquet schema lacks nullable column support and the converter reads the zero-value from `guregu/null` wrappers.

4. **Semantic distinction is real**: The `addUnmuxed` vs `addMuxed` helper split at effects.go:187-198 is deliberate design evidence that the field's presence/absence carries meaning. The codebase treats "no muxed address" and "has muxed address" as semantically distinct states.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing)
- **Setup**: Create an `EffectOutput` with `AddressMuxed: null.String{}` (simulating an unmuxed account effect, as `addUnmuxed` does). Create a second with `AddressMuxed: null.StringFrom("MAAA...")` (simulating a muxed account effect).
- **Steps**: (1) JSON-marshal both outputs — assert the unmuxed one has `address_muxed` absent/null and the muxed one has the M... string. (2) Call `.ToParquet()` on both — extract `AddressMuxed` from the resulting `EffectOutputParquet`.
- **Assertion**: Assert that the unmuxed Parquet row has `AddressMuxed == ""` while the JSON row has null, proving the null collapse. Assert that JSON distinguishes the two rows but Parquet does not (both unmuxed "" and a hypothetical empty-string address would be indistinguishable).

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestEffectAddressMuxedNullCollapse"
**Test Language**: Go

### Demonstration

The test constructs two `EffectOutput` values — one with `AddressMuxed: null.String{}` (unmuxed, as `addUnmuxed` produces) and one with `AddressMuxed: null.StringFrom("MAAZI...")` (muxed). JSON marshaling correctly represents the unmuxed case as `null`, preserving the distinction. However, `ToParquet()` reads `.String` from the `null.String` wrapper, collapsing the null value to `""`. This proves that Parquet output loses the null/present distinction for `address_muxed` on the vast majority of effect rows.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/guregu/null"
)

// TestEffectAddressMuxedNullCollapse demonstrates that the Parquet conversion
// collapses a null (missing) address_muxed into an empty string, making it
// indistinguishable from a genuinely empty value.
func TestEffectAddressMuxedNullCollapse(t *testing.T) {
	// 1. Create an EffectOutput simulating an unmuxed account (addUnmuxed path)
	unmuxedEffect := EffectOutput{
		Address:      "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
		AddressMuxed: null.String{}, // Valid=false, String="" — as addUnmuxed sets it
		OperationID:  12345,
		Type:         1,
		TypeString:   "account_created",
	}

	// 2. Create an EffectOutput simulating a muxed account (addMuxed path)
	muxedAddress := "MAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCAAAAAAAAAPCICBKU"
	muxedEffect := EffectOutput{
		Address:      "GAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCCWN7",
		AddressMuxed: null.StringFrom(muxedAddress), // Valid=true — as addMuxed sets it
		OperationID:  12346,
		Type:         1,
		TypeString:   "account_created",
	}

	// 3. Verify JSON serialization correctly distinguishes the two cases
	unmuxedJSON, err := json.Marshal(unmuxedEffect)
	if err != nil {
		t.Fatalf("failed to marshal unmuxed effect: %v", err)
	}
	muxedJSON, err := json.Marshal(muxedEffect)
	if err != nil {
		t.Fatalf("failed to marshal muxed effect: %v", err)
	}

	// JSON for unmuxed should have address_muxed as null (null.String marshals as JSON null when Valid=false)
	var unmuxedMap map[string]interface{}
	if err := json.Unmarshal(unmuxedJSON, &unmuxedMap); err != nil {
		t.Fatalf("failed to unmarshal unmuxed JSON: %v", err)
	}
	unmuxedVal, exists := unmuxedMap["address_muxed"]
	if !exists {
		t.Fatalf("JSON unmuxed: expected address_muxed to be present as null, but it was absent")
	}
	if unmuxedVal != nil {
		t.Errorf("JSON unmuxed: expected address_muxed to be null, got %v", unmuxedVal)
	}

	// JSON for muxed should contain the M... address
	var muxedMap map[string]interface{}
	if err := json.Unmarshal(muxedJSON, &muxedMap); err != nil {
		t.Fatalf("failed to unmarshal muxed JSON: %v", err)
	}
	if val, exists := muxedMap["address_muxed"]; !exists || val != muxedAddress {
		t.Errorf("JSON muxed: expected address_muxed=%q, got %v", muxedAddress, val)
	}

	// 4. Convert both to Parquet and check the AddressMuxed field
	unmuxedParquet := unmuxedEffect.ToParquet().(EffectOutputParquet)
	muxedParquet := muxedEffect.ToParquet().(EffectOutputParquet)

	// The muxed Parquet row should have the M... address — this is correct
	if muxedParquet.AddressMuxed != muxedAddress {
		t.Errorf("Parquet muxed: expected AddressMuxed=%q, got %q", muxedAddress, muxedParquet.AddressMuxed)
	}

	// BUG: The unmuxed Parquet row has AddressMuxed="" instead of being absent/null.
	// This proves null collapse: a missing muxed address becomes an empty string.
	if unmuxedParquet.AddressMuxed != "" {
		t.Fatalf("Unexpected: unmuxed Parquet AddressMuxed is %q, expected empty string from null collapse", unmuxedParquet.AddressMuxed)
	}

	// The critical proof: JSON distinguishes unmuxed (absent) from muxed (present),
	// but Parquet collapses unmuxed to "" — making it indistinguishable from any
	// other row where AddressMuxed happens to be an empty string.
	t.Logf("BUG CONFIRMED: JSON correctly omits address_muxed for unmuxed accounts")
	t.Logf("BUG CONFIRMED: Parquet writes address_muxed=\"\" for unmuxed accounts (null collapse)")
	t.Logf("Unmuxed Parquet AddressMuxed = %q (should be null/absent, but is empty string)", unmuxedParquet.AddressMuxed)
	t.Logf("Muxed Parquet AddressMuxed = %q (correct)", muxedParquet.AddressMuxed)
}
```

### Test Output

```
=== RUN   TestEffectAddressMuxedNullCollapse
    data_integrity_poc_test.go:83: BUG CONFIRMED: JSON correctly omits address_muxed for unmuxed accounts
    data_integrity_poc_test.go:84: BUG CONFIRMED: Parquet writes address_muxed="" for unmuxed accounts (null collapse)
    data_integrity_poc_test.go:85: Unmuxed Parquet AddressMuxed = "" (should be null/absent, but is empty string)
    data_integrity_poc_test.go:86: Muxed Parquet AddressMuxed = "MAAZI4TCR3TY5OJHCTJC2A4QSY6CJWJH5IAJTGKIN2ER7LBNVKOCAAAAAAAAAPCICBKU" (correct)
--- PASS: TestEffectAddressMuxedNullCollapse (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.771s
```
