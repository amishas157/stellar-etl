# H004: Effects Parquet rewrites null `address_muxed` to empty strings

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Effects involving classic accounts should preserve `address_muxed` as null/absent, while effects involving muxed accounts should preserve the real `M...` address. The Parquet export should keep that distinction instead of serializing unmuxed rows as `address_muxed=""`.

## Mechanism

The effects helpers explicitly model muxed-address presence with `null.String`: `addUnmuxed()` passes `null.String{}` and `addMuxed()` sets `null.StringFrom(address.Address())` only for muxed accounts. But `EffectOutputParquet` narrows `AddressMuxed` to a required `string`, and `EffectOutput.ToParquet()` writes `eo.AddressMuxed.String`, converting every unmuxed effect row into `""`. The Parquet path therefore erases the difference between “this effect had no muxed address” and “this string column contains some present-but-empty sentinel.”

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

The base `address` column still survives, so primary effect attribution remains possible. But consumers that rely on `address_muxed` specifically lose the JSON-layer distinction between “not muxed” and “present but empty,” which is still silent structural corruption of that field.
