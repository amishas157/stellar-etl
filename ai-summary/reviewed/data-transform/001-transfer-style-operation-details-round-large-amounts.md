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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `extractOperationDetails()` in `internal/transform/operation.go` which is the function called by `TransformOperation()` (line 54) to populate the exported `OperationOutput.OperationDetails` map. Confirmed that CreateAccount (line 600), Payment (line 614), CreateClaimableBalance (line 884), and Clawback (line 932) all store monetary amounts as `float64` via `utils.ConvertStroopValueToReal()`. The output map is `map[string]interface{}` (schema.go line 142), which is NOT constrained to float64. The same function already uses `amount.String()` for `destination_min` (line 674), proving string representations work in this context. The sibling `transactionOperationWrapper.Details()` (line 1364) uses `amount.String()` for all the same fields but is never called — only `extractOperationDetails()` feeds the export path.

### Code Paths Examined

- `internal/transform/operation.go:TransformOperation:54` — calls `extractOperationDetails()`, not `transactionOperationWrapper.Details()`
- `internal/transform/operation.go:extractOperationDetails:600` — `details["starting_balance"] = utils.ConvertStroopValueToReal(op.StartingBalance)` — float64
- `internal/transform/operation.go:extractOperationDetails:614` — `details["amount"] = utils.ConvertStroopValueToReal(op.Amount)` — float64 (Payment)
- `internal/transform/operation.go:extractOperationDetails:884` — `details["amount"] = utils.ConvertStroopValueToReal(op.Amount)` — float64 (CreateClaimableBalance)
- `internal/transform/operation.go:extractOperationDetails:932` — `details["amount"] = utils.ConvertStroopValueToReal(op.Amount)` — float64 (Clawback)
- `internal/transform/operation.go:extractOperationDetails:674` — `details["destination_min"] = amount.String(op.DestMin)` — exact string in same function, proving the map accepts strings
- `internal/transform/operation.go:transactionOperationWrapper.Details:1372` — `details["starting_balance"] = amount.String(op.StartingBalance)` — exact string in sibling
- `internal/transform/operation.go:transactionOperationWrapper.Details:1377` — `details["amount"] = amount.String(op.Amount)` — exact string in sibling
- `internal/transform/schema.go:142` — `OperationDetails map[string]interface{}` — untyped map, not constrained to float64
- `internal/utils/main.go:ConvertStroopValueToReal:85-87` — `big.NewRat(int64(input), int64(10000000)).Float64()` — best possible float64, but still lossy for large values

### Findings

The hypothesis is correct. `extractOperationDetails()` uses `ConvertStroopValueToReal()` which produces `float64` values for monetary operation detail fields. While `ConvertStroopValueToReal()` produces the **best possible** float64 conversion (via `big.NewRat().Float64()`), the output container `map[string]interface{}` is not constrained to float64. Three key factors distinguish this from the previously rejected account/trade/trustline hypotheses (034, 036) which had typed `float64` schema fields:

1. **Untyped container**: `OperationOutput.OperationDetails` is `map[string]interface{}`, not `float64`. Strings are a valid alternative with no schema change needed.

2. **Intra-function inconsistency**: Within the same `extractOperationDetails()` function, `destination_min` (line 674) already uses `amount.String()` for an exact decimal string while sibling monetary fields use `ConvertStroopValueToReal()` for lossy float64. This proves the map already accepts both types.

3. **Sibling function demonstrates the fix**: `transactionOperationWrapper.Details()` (line 1364+) uses `amount.String()` for every monetary field that `extractOperationDetails()` converts to float64. This is a complete, correct reference implementation in the same file — it simply isn't wired into the export path.

For stroop values exceeding ~10^16 (amounts above ~900M XLM after scaling), adjacent stroop values collide when represented as float64. The confirmed findings 016 and 017 established that precision loss in monetary fields is a Critical data-correctness bug when a better representation is available. This case is analogous: a better representation (`amount.String()`) is trivially available, already used by sibling code in the same file, and the output container supports it without modification.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (or `internal/transform/data_integrity_poc_test.go` if it exists)
- **Setup**: Build a `CreateAccount` operation with `StartingBalance = 90071992547409931` (exceeds float64 exact integer range after scaling). Create two operations differing by 1 stroop: `90071992547409930` and `90071992547409931`. Wrap each in a successful `LedgerTransaction` with appropriate result metadata.
- **Steps**: Call `TransformOperation()` for each operation and extract `OperationDetails["starting_balance"]` from each result.
- **Assertion**: Assert that the two exported `starting_balance` values are NOT equal (they should differ by 0.0000001). Under the current code, both will produce the same float64, demonstrating the precision loss. Additionally, verify that `amount.String(xdr.Int64(90071992547409931))` produces the exact string `"9007199254.7409931"` while the exported float64 produces a different decimal representation.
