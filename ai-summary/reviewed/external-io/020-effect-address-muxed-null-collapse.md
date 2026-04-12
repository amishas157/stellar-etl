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
