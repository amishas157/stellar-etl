# H001: Failed `path_payment_strict_receive` rows flip `details.source_amount` from number to string

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.source_amount` should keep a stable JSON type for all `path_payment_strict_receive` rows. If a failed path payment has no realized send amount, the exporter should emit either a numeric `0` or `null`, not a quoted decimal string, so successful and failed rows can be consumed with the same schema.

## Mechanism

`extractOperationDetails()` initializes `details["source_amount"]` with `amount.String(0)`, which is a string, and only overwrites it with `utils.ConvertStroopValueToReal(...)` when the transaction succeeds. `TransformOperation()` then writes that mixed-typed map directly into both `OperationDetails` and `OperationDetailsJSON`, so failed rows serialize `"source_amount":"0.0000000"` while successful rows serialize `"source_amount":894.6764349`.

## Trigger

1. Export a ledger containing a failed `path_payment_strict_receive` operation.
2. Compare its `history_operations.details.source_amount` field with a successful `path_payment_strict_receive` row.
3. Observe that the failed row emits a quoted decimal string while the successful row emits a JSON number for the same key.

## Target Code

- `internal/transform/operation.go:extractOperationDetails:619-656` — seeds `source_amount` with `amount.String(0)` and conditionally overwrites it only on success
- `internal/transform/operation.go:TransformOperation:29-57` — forwards the mixed-typed details map into exported JSON
- `internal/transform/operation_test.go:1099-1129` — asserts the successful branch exports numeric `source_amount`, but does not cover the failed branch

## Evidence

The live exporter's strict-receive branch uses two different Go types for the same key: a string sentinel on the default path and a `float64` on the success path. The successful-path test fixture confirms the repository already treats `source_amount` as a number in at least one branch of this operation family.

## Anti-Evidence

`OperationDetails` is a free-form `map[string]interface{}`, so the project may tolerate some per-row type variation. Reviewer may also decide that the upstream Horizon convention of defaulting failed path-payment outcomes to zero makes the quoted zero acceptable even though the successful branch is numeric.
