# H004: Effects Parquet rewrites null `address_muxed` to empty strings

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Effects involving classic accounts should preserve `address_muxed` as null/absent, while effects involving muxed accounts should preserve the real `M...` address. The Parquet export should keep that distinction instead of serializing unmuxed rows as `address_muxed=""`.

## Mechanism

The effects helpers explicitly model muxed-address presence with `null.String`: `addUnmuxed()` passes `null.String{}` and `addMuxed()` sets `null.StringFrom(address.Address())` only for muxed accounts. But `EffectOutputParquet` narrows `AddressMuxed` to a required `string`, and `EffectOutput.ToParquet()` writes `eo.AddressMuxed.String`, converting every unmuxed effect row into `""`. The Parquet path therefore erases the difference between "this effect had no muxed address" and "this string column contains some present-but-empty sentinel."

## Trigger

Run `export_effects --write-parquet` on any ledger range that includes ordinary classic-account effects alongside muxed-account effects. JSON preserves the distinction between missing and populated `address_muxed`, while Parquet emits empty strings for the unmuxed rows.

## Target Code

- `internal/transform/effects.go:effectsWrapper.addUnmuxed:187-188` — classic-account effects use an invalid `null.String`
- `internal/transform/effects.go:effectsWrapper.addMuxed:191-197` — muxed-account effects populate a real muxed address
- `internal/transform/schema.go:EffectOutput:360-372` — JSON schema models `address_muxed` as `null.String`
- `internal/transform/schema_parquet.go:EffectOutputParquet:246-258` — Parquet schema narrows `AddressMuxed` to `string`
- `internal/transform/parquet_converter.go:EffectOutput.ToParquet:277-289` — converter serializes `eo.AddressMuxed.String`
- `cmd/export_effects.go:33-68` — live parquet export path writes `EffectOutputParquet`

## Evidence

`addUnmuxed()` in `effects.go:187-188` hard-codes `null.String{}` for non-muxed accounts, while `addMuxed()` in `effects.go:191-197` populates the field only when the account type is `MuxedEd25519`. The fixtures in `internal/transform/effects_test.go:675-791` already assert real non-empty `AddressMuxed` values for muxed-account cases, proving the field is meant to distinguish two states before Parquet conversion.

## Anti-Evidence

The base `address` column still survives, so primary effect attribution remains possible. But consumers that rely on `address_muxed` specifically lose the JSON-layer distinction between "not muxed" and "present but empty," which is still silent structural corruption of that field.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced from `addUnmuxed()` through `add()` into the `EffectOutput` struct, then through `ToParquet()` into `EffectOutputParquet`. The JSON schema uses `null.String` (from `guregu/null`) which marshals `null.String{Valid: false}` to JSON `null` and `null.StringFrom("M...")` to `"M..."`, preserving the absent-vs-present distinction. The Parquet schema uses plain `string`, and the converter reads `.String` on the `null.String` wrapper, which returns `""` for invalid (null) values. This is the same class of bug as the confirmed success case 003 (`min_account_sequence` null.Int → int64 collapse) but affects a different entity type and field.

### Code Paths Examined

- `internal/transform/effects.go:187-188` — `addUnmuxed()` passes `null.String{}` (Valid=false, String=""), confirming classic-account effects carry a semantically-null muxed address
- `internal/transform/effects.go:191-197` — `addMuxed()` only populates `null.StringFrom(address.Address())` when account type is `MuxedEd25519`; otherwise `addressMuxed` remains zero-valued `null.String{}`
- `internal/transform/effects.go:176-184` — `add()` stores the `null.String` directly into `EffectOutput.AddressMuxed`
- `internal/transform/schema.go:363` — `AddressMuxed null.String` with `json:"address_muxed,omitempty"` tag; `null.String{Valid:false}` marshals to JSON `null` (not omitted — `omitempty` doesn't skip struct zero values)
- `internal/transform/schema_parquet.go:249` — `AddressMuxed string` — no nullable wrapper, plain Go string
- `internal/transform/parquet_converter.go:280` — `eo.AddressMuxed.String` reads the underlying string field directly, returning `""` for null values without checking `.Valid`

### Findings

The bug follows the identical pattern as the confirmed success 003 (transaction `min_account_sequence` null.Int → int64). In JSON output, classic-account effects emit `"address_muxed": null` while muxed-account effects emit `"address_muxed": "M..."`. In Parquet output, classic-account effects emit `address_muxed = ""` — collapsing the null marker into an empty string. Downstream consumers reading Parquet cannot distinguish "this effect was on a classic account" from "this effect was on a muxed account with an empty-string address" (the latter is unrealistic but the semantic distinction is lost).

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (or `internal/transform/effects_test.go`)
- **Setup**: Construct two `EffectOutput` values: one with `AddressMuxed: null.String{}` (classic account) and one with `AddressMuxed: null.StringFrom("MAAAA...")` (muxed account)
- **Steps**: (1) JSON-marshal both and verify the first produces `"address_muxed":null` while the second produces `"address_muxed":"MAAAA..."`. (2) Call `.ToParquet()` on both and cast to `EffectOutputParquet`. (3) Compare `AddressMuxed` fields in the Parquet structs.
- **Assertion**: Assert that the Parquet `AddressMuxed` for the classic-account effect is `""` while the JSON marshaling preserves `null`, demonstrating the information loss. The two Parquet rows should differ, but the null-vs-empty distinction that exists in JSON is collapsed in Parquet.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-10
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestParquetCollapsesNullAddressMuxedToEmptyString"
**Test Language**: Go

### Demonstration

The test constructs two `EffectOutput` values — one with `AddressMuxed: null.String{}` (classic account, no muxed address) and one with `AddressMuxed: null.StringFrom("MAAAA...")` (muxed account). JSON marshaling preserves the null-vs-present distinction correctly. However, calling `.ToParquet()` on the classic-account effect produces `AddressMuxed = ""` — collapsing the semantically-null field into an empty string, making it impossible for Parquet consumers to distinguish "no muxed address" from "empty muxed address".

### Test Body

```go
func TestParquetCollapsesNullAddressMuxedToEmptyString(t *testing.T) {
	muxedAddr := "MAAAAAAAAAAAAAB7BQ2L7E5NBWMN77UJ2LQMLJFSG7MFQOO6BLSW4MRWW33XPUZ6HANPT6EHJBPK"

	// Classic-account effect: AddressMuxed should be null (no muxed address)
	classicEffect := EffectOutput{
		Address:      "GABC",
		AddressMuxed: null.String{}, // Valid=false — no muxed address
		OperationID:  12345,
		Type:         1,
		TypeString:   "account_created",
	}

	// Muxed-account effect: AddressMuxed should carry the real M... address
	muxedEffect := EffectOutput{
		Address:      "GABC",
		AddressMuxed: null.StringFrom(muxedAddr), // Valid=true — real muxed address
		OperationID:  12346,
		Type:         1,
		TypeString:   "account_created",
	}

	t.Run("JSON_preserves_null_vs_present", func(t *testing.T) {
		classicJSON, err := json.Marshal(classicEffect.AddressMuxed)
		if err != nil {
			t.Fatalf("failed to marshal classic AddressMuxed: %v", err)
		}
		if string(classicJSON) != "null" {
			t.Fatalf("expected classic AddressMuxed JSON to be null, got %s", string(classicJSON))
		}

		muxedJSON, err := json.Marshal(muxedEffect.AddressMuxed)
		if err != nil {
			t.Fatalf("failed to marshal muxed AddressMuxed: %v", err)
		}
		if string(muxedJSON) != `"`+muxedAddr+`"` {
			t.Fatalf("expected muxed AddressMuxed JSON to be %q, got %s", muxedAddr, string(muxedJSON))
		}
	})

	t.Run("Parquet_collapses_null_to_empty_string", func(t *testing.T) {
		classicParquet := classicEffect.ToParquet().(EffectOutputParquet)
		muxedParquet := muxedEffect.ToParquet().(EffectOutputParquet)

		// Muxed-account effect should preserve the address in Parquet
		if muxedParquet.AddressMuxed != muxedAddr {
			t.Errorf("muxed-account Parquet AddressMuxed should be %s, got %s",
				muxedAddr, muxedParquet.AddressMuxed)
		}

		// Classic-account effect: Parquet collapses null to ""
		if classicParquet.AddressMuxed != "" {
			t.Fatalf("expected classic Parquet AddressMuxed to be empty string, got %q",
				classicParquet.AddressMuxed)
		}

		// The bug: JSON has null, Parquet has "" — null semantics are lost
		if !classicEffect.AddressMuxed.Valid && classicParquet.AddressMuxed == "" {
			t.Errorf("NULL COLLAPSE CONFIRMED: Effect AddressMuxed JSON is null (Valid=%v), "+
				"but Parquet AddressMuxed is %q — consumers cannot distinguish "+
				"'no muxed address' from 'empty muxed address'",
				classicEffect.AddressMuxed.Valid, classicParquet.AddressMuxed)
		}
	})
}
```

### Test Output

```
=== RUN   TestParquetCollapsesNullAddressMuxedToEmptyString
=== RUN   TestParquetCollapsesNullAddressMuxedToEmptyString/JSON_preserves_null_vs_present
=== RUN   TestParquetCollapsesNullAddressMuxedToEmptyString/Parquet_collapses_null_to_empty_string
    data_integrity_poc_test.go:180: NULL COLLAPSE CONFIRMED: Effect AddressMuxed JSON is null (Valid=false), but Parquet AddressMuxed is "" — consumers cannot distinguish 'no muxed address' from 'empty muxed address'
--- FAIL: TestParquetCollapsesNullAddressMuxedToEmptyString (0.00s)
    --- PASS: TestParquetCollapsesNullAddressMuxedToEmptyString/JSON_preserves_null_vs_present (0.00s)
    --- FAIL: TestParquetCollapsesNullAddressMuxedToEmptyString/Parquet_collapses_null_to_empty_string (0.00s)
FAIL
```
