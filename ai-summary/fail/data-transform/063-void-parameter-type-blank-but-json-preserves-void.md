# H063: `ScvVoid` invoke parameters lose the convenience type label, but canonical JSON still preserves the type

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Low
**Impact**: helper-field presentation
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For an `invoke_host_function` argument with XDR type `ScValTypeScvVoid`, the human-readable helper fields (`parameters` / `parameters_decoded`) would ideally expose an explicit non-empty type label rather than an empty string. That would make the convenience view consistent with the rest of the `ScVal` arms.

## Mechanism

`serializeParameters()` uses `param.ArmForSwitch(int32(param.Type))` to fill the `"type"` field. The generated XDR helper returns `("", true)` for `ScValTypeScvVoid`, so the convenience maps end up with a blank type even though the raw bytes and decoded value are still preserved.

## Trigger

Export an `invoke_host_function` operation whose argument list or constructor arguments include a `ScVal{Type: ScValTypeScvVoid}` value. The `parameters` and `parameters_decoded` helper arrays will contain an entry with `"type": ""`.

## Target Code

- `internal/transform/operation.go:2247-2270` — `serializeParameters()` copies the arm name directly into the helper maps
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250219002427-fe9b5ecd408e/xdr/xdr_generated.go:58131-58137` — generated `ArmForSwitch()` returns an empty string for `ScValTypeScvVoid`
- `internal/transform/contract_events.go:96-137` — the canonical JSON serialization path still preserves the raw XDR and decoded JSON form

## Evidence

Local inspection confirmed that `ScvVoid` marshals as `00000001`, `ScVal.String()` renders it as `(void)`, and `xdrjson.Decode(xdrjson.ScVal, raw)` returns `"void"`. The blank type label is therefore confined to the helper maps built by `serializeParameters()`.

## Anti-Evidence

The same operation details also export `parameters_json` and `parameters_json_decoded`, which preserve both the raw XDR payload and an unambiguous decoded `"void"` value. Downstream consumers that need correctness can recover the type from those canonical fields even when the convenience `"type"` label is blank.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

This only degrades a secondary convenience projection. The canonical operation-detail exports (`parameters_json` and `parameters_json_decoded`) still preserve the `void` type unambiguously, so this does not rise to a meaningful data-integrity bug under the current objective.

### Lesson Learned

When a lossy helper field sits beside a canonical raw/decoded export of the same value, prefer the canonical path when deciding whether the bug materially corrupts downstream data. Small presentation defects in duplicate convenience fields are not enough by themselves.
