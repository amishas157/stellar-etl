# H003: Mirrored path-payment limit fields use different JSON types

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The caller-supplied limit field for the two mirrored path-payment operations should use one stable representation across the export surface. `path_payment_strict_receive.details.source_max` and `path_payment_strict_send.details.destination_min` are the two user-provided bounds that cap how much value the path payment may consume or deliver, so they should both serialize as exact decimal strings or both serialize as numbers.

## Mechanism

`extractOperationDetails()` exports `strict_receive.source_max` through `utils.ConvertStroopValueToReal(op.SendMax)` but exports the mirrored `strict_send.destination_min` through `amount.String(op.DestMin)`. That means the same conceptual limit field family changes JSON type between operation variants: one branch emits a `float64`, the other an exact string.

## Trigger

1. Export one successful `path_payment_strict_receive` row and one successful `path_payment_strict_send` row.
2. Compare `details.source_max` against `details.destination_min`.
3. Observe that the strict-receive limit is a JSON number while the strict-send limit is a quoted decimal string, despite both fields representing the user-specified path-payment bound.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:631-674` — uses `ConvertStroopValueToReal()` for `source_max` and `amount.String()` for `destination_min`
- `internal/transform/operation_test.go:1103-1113` — successful strict-receive fixture asserts numeric `source_max`
- `internal/transform/operation_test.go:1449-1455` — successful strict-send fixture asserts string `destination_min`

## Evidence

Both mirrored branches live in the same exporter function, so the divergent typing is not coming from separate subsystems or a dead fallback path. The checked-in fixtures explicitly show the current output contract already diverges between the two path-payment variants.

## Anti-Evidence

The field names are different (`source_max` vs `destination_min`), so reviewer may decide they are not required to share a schema despite their mirrored semantics. The tests also lock in the current divergent behavior, which may be treated as contract rather than corruption.
