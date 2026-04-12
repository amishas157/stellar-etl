# H065: `contract_event_xdr` should serialize raw `ContractEvent`

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If the `contract_event_xdr` column is meant to store the raw Stellar `ContractEvent` payload, then the exported base64 should be the serialization of the event itself, without any surrounding `DiagnosticEvent` wrapper. Consumers should be able to decode the blob directly as the same XDR object implied by the column name.

## Mechanism

`parseDiagnosticEvent()` builds all contract-event rows from a `DiagnosticEvent` wrapper and then serializes that wrapper into `ContractEventXDR` with `xdr.MarshalBase64(diagnosticEvent)`. That looked like a shape bug because the exported blob includes wrapper-level metadata (`InSuccessfulContractCall`) rather than only the embedded `ContractEvent`.

## Trigger

Export any transaction that produces contract events or diagnostic events, then decode `contract_event_xdr` and compare it against the raw `ContractEvent` bytes for the same row.

## Target Code

- `internal/transform/contract_events.go:parseDiagnosticEvent:194-239` — serializes `diagnosticEvent` into `ContractEventXDR`
- `internal/transform/contract_events_test.go:58-145` — asserts the exact current `ContractEventXDR` base64 strings

## Evidence

The implementation unquestionably marshals the wrapper, not `diagnosticEvent.Event`. The resulting bytes therefore encode more than just the raw event body that the field name seems to suggest.

## Anti-Evidence

Unit tests explicitly bless the current `ContractEventXDR` values, and the whole transform intentionally flattens transaction events, operation events, and diagnostic events through a shared `DiagnosticEvent` representation before export. Within that design, the wrapper blob is consistent with the actual parsed source object for the row.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The codebase intentionally normalizes all event streams through `DiagnosticEvent` before serialization, and the regression tests lock in that wrapper-level blob. This is a naming/interpretation concern, not a confirmed wrong-value export under the repository's current contract.

### Lesson Learned

When an export intentionally flattens heterogeneous XDR streams into a shared wrapper type, wrapper serialization may be the designed payload even if the column name sounds narrower. Test fixtures that assert the exact blob are strong evidence against treating the format choice as fresh corruption.
