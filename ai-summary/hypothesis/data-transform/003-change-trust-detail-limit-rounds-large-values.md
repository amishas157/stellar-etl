# H003: Change-trust detail `limit` rounds large trustline limits

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.limit` for a successful `change_trust` operation should preserve the exact 7-decimal trustline limit. A limit of `90071992547409931` stroops should export as `9007199254.7409931`, and adjacent 1-stroop limits should remain distinct.

## Mechanism

`extractOperationDetails()` converts `ChangeTrustOp.Limit` through `utils.ConvertStroopValueToReal()` and stores the rounded `float64` in the untyped detail map. The sibling `transactionOperationWrapper.Details()` path already exports the same field with `amount.String(op.Limit)`, so this is not an unavoidable schema constraint: the operation export is choosing a lossy encoding where an exact decimal string already exists.

## Trigger

Process a successful `change_trust` operation with a very large limit, such as `Limit = 90071992547409931`, then compare the exported `details.limit` against `amount.String(op.Limit)` or against a second operation with `Limit = 90071992547409932`. The JSON export will round away the 1-stroop distinction.

## Target Code

- `internal/transform/operation.go:800-820` — `change_trust.limit` is emitted through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1495-1506` — sibling wrapper emits `details["limit"] = amount.String(op.Limit)`
- `internal/transform/schema.go:136-150` — operation details are stored in a dynamic map, so exact strings are allowed

## Evidence

This branch is a clean A/B comparison inside the same file: one implementation path uses `float64`, the other uses the exact decimal string. That makes the suspected corruption concrete and testable, rather than a generic complaint about float64 types elsewhere in the repository.

## Anti-Evidence

The checked-in operation fixture uses `Limit = 500000000000000000` and expects `50000000000.0`, so historical tests currently codify the numeric form for a small-enough value. A reviewer may treat that as intentional formatting, but the file itself already contains a precision-preserving alternative implementation for the same field.
