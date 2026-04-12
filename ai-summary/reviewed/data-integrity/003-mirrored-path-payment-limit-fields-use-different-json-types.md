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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

In `extractOperationDetails()` (the live export path), every amount and limit field uses `utils.ConvertStroopValueToReal()` to produce `float64` values — except `destination_min` on line 674, which uses `amount.String(op.DestMin)` to produce a string. This makes `destination_min` the sole outlier among ~20 amount fields in the function. The unreachable `transactionOperationWrapper.Details()` implementation (lines 1386, 1409) uses `amount.String()` consistently for both fields, suggesting `destination_min` in the live path is a copy-paste artifact that was never updated.

### Code Paths Examined

- `internal/transform/operation.go:619-658` — strict-receive branch: `source_max` assigned via `ConvertStroopValueToReal(op.SendMax)` → float64
- `internal/transform/operation.go:660-699` — strict-send branch: `destination_min` assigned via `amount.String(op.DestMin)` → string
- `internal/transform/operation.go:1379-1416` — unreachable `transactionOperationWrapper.Details()`: both `source_max` and `destination_min` use `amount.String()` consistently (both strings)
- `internal/utils/main.go:84-88` — `ConvertStroopValueToReal` divides xdr.Int64 by 1e7, returns float64
- `internal/transform/operation_test.go:1107` — fixture asserts `source_max: 895.14959` (float64 literal)
- `internal/transform/operation_test.go:1453` — fixture asserts `destination_min: "428.0460538"` (string literal)

### Findings

1. **Type inconsistency confirmed**: `destination_min` is the only amount/limit field in `extractOperationDetails()` that uses `amount.String()` instead of `ConvertStroopValueToReal()`. All other amount fields (~20 occurrences on lines 600-1061) produce float64.

2. **Downstream impact**: JSON output contains a float64 for `source_max` (e.g., `895.14959`) and a string for `destination_min` (e.g., `"428.0460538"`). Downstream consumers doing cross-operation-type analytics on path payment limits must handle heterogeneous types.

3. **Copy-paste origin**: The unreachable `transactionOperationWrapper.Details()` uses `amount.String()` for both fields consistently. The live `extractOperationDetails()` function was updated to use `ConvertStroopValueToReal()` for most fields, but `destination_min` was missed — it still uses the old `amount.String()` pattern.

4. **Severity downgrade rationale**: Reduced from High to Medium because the data VALUE is correct in both representations — only the JSON type differs. The fields appear in different operation types with different names, so no single row has conflicting types. However, the inconsistency breaks the function's own established pattern and creates a schema divergence that downstream systems must handle.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Use the existing test infrastructure with `makeOperationTestInput()` for both `PathPaymentStrictReceive` and `PathPaymentStrictSend` operation types.
- **Steps**: Call `extractOperationDetails()` for a successful `PathPaymentStrictReceive` and a successful `PathPaymentStrictSend`. Inspect the Go types of `details["source_max"]` and `details["destination_min"]`.
- **Assertion**: Assert that `reflect.TypeOf(details["source_max"])` is `float64` while `reflect.TypeOf(details["destination_min"])` is `string`, confirming the type divergence. Alternatively, JSON-marshal both detail maps and verify that `source_max` appears as an unquoted number while `destination_min` appears as a quoted string.
