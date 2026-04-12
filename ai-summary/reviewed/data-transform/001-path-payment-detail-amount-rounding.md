# H001: Path-payment detail amounts collapse distinct stroop values

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details` should preserve distinct path-payment amounts and bounds when the on-chain stroop values differ. For successful `path_payment_strict_receive` and `path_payment_strict_send` operations, fields such as `amount`, `source_max`, and `source_amount` should export different values for inputs like `90071992547409930` and `90071992547409931` stroops instead of collapsing them to the same JSON number.

## Mechanism

`extractOperationDetails()` converts the strict-receive `amount`, `source_max`, and successful `source_amount`, plus the strict-send `source_amount` and successful `amount`, through `utils.ConvertStroopValueToReal()`, which rounds to the nearest `float64`. The same `details` map already carries exact decimal strings for sibling fields such as `destination_min`, so this output surface can preserve exact decimal text; large path-payment amounts therefore silently lose one-stroop distinctions only because these branches pick the lossy conversion path.

## Trigger

1. Construct a successful `path_payment_strict_receive` with `DestAmount` or `SendMax` equal to `90071992547409930` stroops and another identical operation with `90071992547409931`.
2. Construct a successful `path_payment_strict_send` with `SendAmount` or resulting `DestAmount()` differing by 1 stroop at the same magnitude.
3. Run both through `TransformOperation()` and compare the exported `details.amount`, `details.source_max`, or `details.source_amount` values.

## Target Code

- `internal/transform/operation.go:619-699` — live path-payment detail export uses `utils.ConvertStroopValueToReal()` for path-payment amounts
- `internal/transform/operation.go:1379-1423` — sibling formatter keeps exact decimal strings for the same logical fields
- `internal/transform/schema.go:136-150` — operation details are exported as a heterogenous `map[string]interface{}`

## Evidence

The strict-receive branch stores `amount` and `source_max` as lossy `float64` values and overwrites `source_amount` with another lossy conversion on success. The strict-send branch similarly stores `source_amount` and successful `amount` as `float64`, even though the same file already uses `amount.String(...)` for `destination_min` and for the equivalent fields in the alternate formatter.

## Anti-Evidence

Downstream consumers can still recover exact values from raw XDR blobs elsewhere in the export, and `destination_min` is already exact on strict-send rows. But the `history_operations.details` payload itself is supposed to be a directly consumable decoded view, and these specific fields are wrong on that surface for large amounts.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (prior rejections 034-040 did not consider the live within-function `destination_min` inconsistency)

### Trace Summary

Traced `extractOperationDetails()` for both `PathPaymentStrictReceive` (lines 619-658) and `PathPaymentStrictSend` (lines 660-699) in `internal/transform/operation.go`. Confirmed that `amount`, `source_max`, `source_amount` (on success), and `source_amount`/`amount` (on success) all pass through `utils.ConvertStroopValueToReal()` which returns lossy `float64`. Critically, in the same function and same operation block, `destination_min` on strict-send (line 674) uses `amount.String(op.DestMin)` producing an exact decimal string. The structural analog `source_max` on strict-receive (line 633) uses the lossy float64 path. This within-function inconsistency in the live code distinguishes this finding from the rejected 034-040 group, which only cited dead code (`transactionOperationWrapper.Details()`) as evidence of a better path.

### Code Paths Examined

- `internal/transform/operation.go:619-658` — `PathPaymentStrictReceive`: `amount` (line 631), `source_max` (line 633), and successful `source_amount` (line 655) all use `utils.ConvertStroopValueToReal()` returning lossy float64
- `internal/transform/operation.go:660-699` — `PathPaymentStrictSend`: `source_amount` (line 673) uses `utils.ConvertStroopValueToReal()` (lossy float64), `destination_min` (line 674) uses `amount.String()` (exact string), successful `amount` (line 696) uses `utils.ConvertStroopValueToReal()` (lossy float64)
- `internal/transform/operation.go:1379-1423` — `transactionOperationWrapper.Details()` uses `amount.String()` for all path-payment fields; confirmed dead code (zero callers outside tests)
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal()` uses `big.NewRat(int64(input), 10000000).Float64()`, the best possible float64 conversion but still lossy for values exceeding float64 precision at 7-decimal scale
- `internal/transform/schema.go:136-150` — `OperationOutput.OperationDetails` is `map[string]interface{}`, capable of holding either strings or floats

### Findings

The `destination_min` field on `PathPaymentStrictSend` already uses `amount.String()` in the same live function (`extractOperationDetails()`), same details map, same operation-type block. This proves the map's consumers already handle string-valued amount fields. The use of `ConvertStroopValueToReal()` for `source_amount`, `amount`, and `source_max` is therefore an oversight, not a schema constraint.

Prior rejections (034-040) covered "path-payment settlement amounts" but only cited the dead-code `transactionOperationWrapper.Details()` as evidence of a better path. The fail summary's own framework distinguishes: (a) "a better path exists but is bypassed → VIABLE" vs (c) "goes directly to float64 via best conversion → NOT_VIABLE schema limit." The live `destination_min` inconsistency satisfies criterion (a).

This finding is also analogous to success #019 (transfer-style operation details float64 rounding for `create_account`, `payment`, etc.) but covers the distinct path-payment operation types not included in that finding.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or a new `internal/transform/data_integrity_poc_test.go`)
- **Setup**: Build two `PathPaymentStrictSend` operations with `SendAmount` values of `90071992547409930` and `90071992547409931` stroops. Construct successful transaction results with `DestAmount()` values differing by 1 stroop at the same magnitude.
- **Steps**: Call `TransformOperation()` for each, extract `OperationDetails["source_amount"]` and `OperationDetails["amount"]`.
- **Assertion**: Assert that the two `source_amount` float64 values are equal (demonstrating the collision), and compare against the exact `amount.String()` representations which remain distinct. Also verify that `OperationDetails["destination_min"]` IS a string (demonstrating the within-function inconsistency).
