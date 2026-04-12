# H020: Effect Parquet rewrites missing `address_muxed` to an empty string

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Effects created for ordinary `G...` accounts should preserve the fact that no muxed address exists. The Parquet row should distinguish “no muxed address” from a real `M...` address, matching the JSON export semantics.

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
