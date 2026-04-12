# H024: `serializeScVal()` might drop `ScValTypeScvVoid` as `"n/a"`

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a contract event or contract-data field contains `ScValTypeScvVoid`, the exporter should preserve that legitimate `void` value in both the raw and decoded payload columns instead of replacing it with a placeholder string.

## Mechanism

At first glance, `serializeScVal()` looks like it may only encode SCVals that have a payload arm, leaving arm-less values at the default `"n/a"` sentinel. That would cause valid `void` payloads to export as fake placeholders instead of their real XDR/JSON representations.

## Trigger

1. Feed `serializeScVal()` a legitimate `xdr.ScVal{Type: ScValTypeScvVoid}` through contract-event or contract-data export.
2. Inspect the returned raw and decoded values.
3. Check whether the function leaves them as `"n/a"`.

## Target Code

- `internal/transform/contract_events.go:96-119` — `serializeScVal()` initializes `"n/a"` placeholders and gates encoding on `ArmForSwitch`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:57666-57705` — `ScVal.ArmForSwitch()` marks `ScValTypeScvVoid` as a valid switch arm

## Evidence

The helper starts with `"n/a"` defaults and only overwrites them inside the `ArmForSwitch` branch, which makes arm-less union values look suspicious on a first read.

## Anti-Evidence

`ScVal.ArmForSwitch()` returns `ok=true` for `ScValTypeScvVoid`, and `param.MarshalBinary()` / `xdrjson.Decode()` already know how to encode that type. So the suspected placeholder fallback never triggers for legitimate `void` values.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The guard is not actually excluding `ScvVoid`: the generated XDR union marks it as a valid switch arm even though it has no payload field, so `serializeScVal()` still marshals and decodes it normally.

### Lesson Learned

For XDR unions, an empty arm and an invalid arm are different things. Always inspect the generated `ArmForSwitch()` output before assuming a no-payload type will hit a placeholder fallback.
