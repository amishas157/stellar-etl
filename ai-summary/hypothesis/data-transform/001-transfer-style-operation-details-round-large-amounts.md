# H001: Transfer-style operation details round large stroop amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`history_operations.details` should preserve the exact decimal amount for successful transfer-style operations. For example, a `create_account.starting_balance`, `payment.amount`, `create_claimable_balance.amount`, or `clawback.amount` of `90071992547409931` stroops should export the exact 7-decimal value `9007199254.7409931`, and a 1-stroop change should remain distinguishable in the output.

## Mechanism

`extractOperationDetails()` stores these fields as `float64` via `utils.ConvertStroopValueToReal()`, even though the same package already has exact decimal-string encodings for the same logical fields in `transactionOperationWrapper.Details()`. Because `OperationOutput.OperationDetails` is a `map[string]interface{}`, the export surface is not forced to use `float64`; large valid on-chain amounts therefore collapse to rounded neighbors only because this branch picks a lossy representation.

## Trigger

Process any successful operation of one of these types with a large amount, such as:

1. `CreateAccount.StartingBalance = 90071992547409931`
2. `PaymentOp.Amount = 90071992547409931`
3. `CreateClaimableBalanceOp.Amount = 90071992547409931`
4. `ClawbackOp.Amount = 90071992547409931`

Compare the exported JSON detail field against the exact `amount.String(...)` representation or against a second operation that differs by 1 stroop; the two values will round together.

## Target Code

- `internal/transform/operation.go:590-617` — `create_account` and `payment` detail amounts are converted through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:881-885` — `create_claimable_balance.amount` uses the same float conversion
- `internal/transform/operation.go:924-932` — `clawback.amount` uses the same float conversion
- `internal/transform/operation.go:1368-1378` — sibling wrapper emits exact `amount.String(...)` values for `create_account` and `payment`
- `internal/transform/operation.go:1537-1541` — sibling wrapper emits exact `amount.String(...)` for `create_claimable_balance`
- `internal/transform/operation.go:1579-1583` — sibling wrapper emits exact `amount.String(...)` for `clawback`
- `internal/transform/schema.go:136-150` — operation details are a dynamic `map[string]interface{}`

## Evidence

The same package already demonstrates an exact representation for these fields: `transactionOperationWrapper.Details()` uses `amount.String(...)` rather than `float64`. That makes this different from the typed top-level account/trade schemas rejected in prior investigations: here the map can already carry strings, and the precision loss comes from a specific conversion choice in `extractOperationDetails()`, not from an unavoidable schema type.

## Anti-Evidence

Checked-in `operation_test.go` fixtures only use small values like `2.5` and `35.0`, so the current regression suite reinforces the lossy shape without exercising large-value behavior. A reviewer may argue that numeric JSON is part of the historical contract, but nearby operation-detail fields already mix strings and numbers, so exact decimal strings would not be unprecedented in this payload.
