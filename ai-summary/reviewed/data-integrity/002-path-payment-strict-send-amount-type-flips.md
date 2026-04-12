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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`TransformOperation()` (line 30) calls `extractOperationDetails()` (line 584), which is the live export path for `history_operations`. In the `PathPaymentStrictSend` case (line 660), `details["amount"]` is initialized to `amount.String(0)` (Go `string`, returns `"0.0000000"`), and only overwritten with `utils.ConvertStroopValueToReal(result.DestAmount())` (Go `float64`) when `transaction.Result.Successful()` is true (line 682). The `map[string]interface{}` is stored directly in `OperationOutput.OperationDetails` and serialized via `encoding/json`, which encodes strings as quoted JSON strings and float64s as JSON numbers — producing different JSON types for the same field.

### Code Paths Examined

- `internal/transform/operation.go:TransformOperation:30-54` — constructs `OperationOutput`, delegates to `extractOperationDetails()` at line 54, stores result in `OperationDetails` at line 92
- `internal/transform/operation.go:extractOperationDetails:660-697` — `PathPaymentStrictSend` branch: line 672 sets `details["amount"] = amount.String(0)` (string), line 696 overwrites with `utils.ConvertStroopValueToReal(result.DestAmount())` (float64) only inside `Successful()` guard
- `internal/utils/main.go:ConvertStroopValueToReal:85` — returns `float64`
- `github.com/stellar/go-stellar-sdk/amount:String:134` — returns `string` ("0.0000000" for input 0)
- `internal/transform/operation.go:extractOperationDetails:619-656` — parallel `PathPaymentStrictReceive` branch exhibits the identical pattern for `source_amount` (string default, float64 on success)

### Findings

1. **Type flip confirmed**: For `PathPaymentStrictSend`, `details["amount"]` is Go type `string` for failed transactions and Go type `float64` for successful ones. `encoding/json.Marshal` serializes these as `"amount":"0.0000000"` vs `"amount":43.3404386` respectively — different JSON types for the same field.

2. **Symmetric issue in PathPaymentStrictReceive**: The same pattern exists for `details["source_amount"]` in `PathPaymentStrictReceive` (line 632 `amount.String(0)` → string; line 655 `ConvertStroopValueToReal` → float64). This is a separate hypothesis (H001) already in the pipeline.

3. **Additional type inconsistency**: `destination_min` (strict-send, line 674) always uses `amount.String()` (always string), while the sibling field `source_max` (strict-receive, line 633) always uses `ConvertStroopValueToReal()` (always float64). These analogous fields use different types even in steady state.

4. **The alternate `Details()` method (line 1364) is type-consistent**: It uses `amount.String()` for all amounts in both path payment types, avoiding the type flip. This confirms the inconsistency in `extractOperationDetails()` is unintentional — the codebase has a correct reference implementation that the live exporter diverges from.

5. **No guard prevents this path**: Failed `PathPaymentStrictSend` operations are legitimate on-chain data (e.g., underfunded paths). The `Successful()` check at line 682 is the sole branch controlling the type of `details["amount"]`.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct an `ingest.LedgerTransaction` with a `PathPaymentStrictSend` operation and a non-successful result code (e.g., `PathPaymentStrictSendResultCodePathPaymentStrictSendNoDestination`). Use existing test helpers like `utils.CreateSampleTx()` as a template.
- **Steps**: Call `TransformOperation()` with the failed transaction. Extract `details["amount"]` from the returned `OperationOutput.OperationDetails`.
- **Assertion**: Assert that `details["amount"]` is a `float64` (not a `string`). Currently it will be `string("0.0000000")` — this demonstrates the type flip. For comparison, also test with a successful transaction and confirm `details["amount"]` is `float64` there.
