# H002: Failed `path_payment_strict_send` rows flip `details.amount` from number to string

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.amount` should have one stable JSON type for every `path_payment_strict_send` row. If a failed strict-send payment delivers nothing, the exporter should emit a numeric `0` or `null`, not a quoted decimal string, so downstream consumers can parse the field consistently across success and failure rows.

## Mechanism

The strict-send branch seeds `details["amount"]` with `amount.String(0)` and only overwrites it with `utils.ConvertStroopValueToReal(result.DestAmount())` when the transaction succeeds. Because `TransformOperation()` exports that map without normalizing types, failed rows serialize `"amount":"0.0000000"` while successful rows serialize `"amount":433.4043858`.

## Trigger

1. Export a ledger containing a failed `path_payment_strict_send` operation.
2. Compare `history_operations.details.amount` on that row with a successful strict-send row.
3. Observe that the failed row uses a quoted decimal string while the successful row uses a JSON number for the same field.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:660-697` — initializes `amount` as a string and rewrites it to `float64` only on success
- `internal/transform/operation.go:TransformOperation:29-57` — passes the mixed-typed details map straight into the output struct
- `internal/transform/operation_test.go:1444-1471` — shows the successful branch exports numeric `amount`, leaving the failed branch untested

## Evidence

The active strict-send exporter uses two incompatible Go value types for `details.amount` depending solely on the result code. The checked-in success fixture confirms the field is already consumed as numeric in the success case, so failure rows silently drift from that shape.

## Anti-Evidence

As with other operation-detail keys, the enclosing `map[string]interface{}` gives the implementation latitude to vary types. Reviewer may conclude that a quoted zero is still a readable representation for failed outcome amounts even if it diverges from the success-path number.
