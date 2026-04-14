# H029: `serializeScVal()` / `serializeParameters()` export `"n/a"` for legitimate current `ScVal`s

**Date**: 2026-04-14
**Subsystem**: external-io
**Severity**: Medium
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Valid decoded contract-event payloads and invoke-host-function parameters should export their real XDR bytes plus decoded representations. Placeholder strings such as `"n/a"` should appear only for impossible internal states, not for legitimate on-chain `ScVal`s.

## Mechanism

`serializeScVal()` and `serializeParameters()` both start from `"n/a"` placeholders and only replace them if `ArmForSwitch(...)` succeeds and the value can be marshaled back to XDR. I suspected some present-day `ScVal` types might decode successfully from ledger data but still miss that path, causing silent placeholder output in `contract_events` or operation parameter exports.

## Trigger

Feed present-day contract-event or invoke-host-function metadata containing a valid but supposedly unsupported `ScVal` shape through `TransformContractEvent()` or `TransformOperation()` and inspect the exported decoded/raw payload fields for `"n/a"`.

## Target Code

- `internal/transform/contract_events.go:96-119` — `serializeScVal()` initializes raw/decoded output to `"n/a"`
- `internal/transform/operation.go:2247-2272` — `serializeParameters()` initializes each parameter map to `"n/a"`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:57634-57714` — current `ScVal.ArmForSwitch` covers all current enum arms
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/xdr/xdr_generated.go:58498-58698` — `ScVal.DecodeFrom` rejects invalid switch values and populates the matching arm for valid ones

## Evidence

The production export code really does have a placeholder fallback instead of returning an explicit error, so if a valid decoded `ScVal` could bypass arm selection or re-marshaling it would silently corrupt exported payloads.

## Anti-Evidence

The pinned XDR generator currently enumerates all live `ScVal` arms in `ArmForSwitch`, and the decoder rejects invalid discriminants before the transform layer ever sees them. For legitimate decoded ledger values, the transformer receives a well-formed union with a recognized arm, so the placeholder path is not reachable without an invalid in-memory `ScVal`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-14
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The `"n/a"` fallback is guarding impossible-in-production union states, not normal chain data. Current decoded `ScVal`s already have recognized discriminants and populated arms, so they serialize through the normal path.

### Lesson Learned

Placeholder defaults are only interesting when legitimate decoded XDR can actually miss the normal branch. For generated unions, check both `ArmForSwitch` and `DecodeFrom` before treating fallback strings as a live corruption path.
