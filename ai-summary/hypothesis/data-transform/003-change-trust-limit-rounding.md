# H003: `change_trust.limit` rounds large trustline limits in operation details

**Date**: 2026-04-12
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details.limit` for a successful `change_trust` operation should preserve the exact trustline limit requested on-chain. Two otherwise identical `change_trust` operations whose `Limit` differs by 1 stroop should export different detail values.

## Mechanism

The `change_trust` branch converts `op.Limit` with `utils.ConvertStroopValueToReal()`, so large limits are rounded to the nearest `float64` before they ever reach JSON. The same file's alternate formatter already uses `amount.String(op.Limit)`, and the details payload already contains strings for other numeric fields, so this is another avoidable precision loss on a monetary field.

## Trigger

1. Create two successful `change_trust` operations with the same asset and trustor but limits `90071992547409930` and `90071992547409931` stroops.
2. Run both through `TransformOperation()`.
3. Compare `OperationDetails["limit"]` across the two exports.

## Target Code

- `internal/transform/operation.go:800-821` — live `change_trust` detail export converts `limit` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1495-1506` — sibling formatter keeps the exact `amount.String(op.Limit)` representation
- `internal/transform/schema.go:136-150` — operation details allow exact-string output

## Evidence

The live branch stores `details["limit"]` as a `float64`, while the sibling formatter stores the same field as an exact decimal string. Because `change_trust` limits are XDR `Int64` values, the same large-stroop trigger used for other operation-detail precision bugs applies here.

## Anti-Evidence

The state snapshot table `trust_lines` still preserves the raw integer `trust_line_limit`, so this does not corrupt the current trustline state export. The issue is specifically that the decoded historical operation-details row becomes wrong for high-value trustline-limit changes.
