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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `extractOperationDetails()` in `operation.go` from line 619 through the `PathPaymentStrictReceive` case. Confirmed that line 632 sets `details["source_amount"] = amount.String(0)` which returns Go type `string` (value `"0.0000000"`), while line 655 inside the `transaction.Result.Successful()` guard overwrites with `utils.ConvertStroopValueToReal(result.SendAmount())` which returns Go type `float64`. The `map[string]interface{}` is passed directly into `OperationOutput.OperationDetails` and `OperationDetailsJSON` at lines 92 and 97, so JSON serialization preserves the Go type distinction. The identical pattern exists for `details["amount"]` in `PathPaymentStrictSend` (lines 672 vs 696).

### Code Paths Examined

- `internal/transform/operation.go:extractOperationDetails:619-656` — Confirmed: line 632 assigns `amount.String(0)` (string `"0.0000000"`), line 655 assigns `utils.ConvertStroopValueToReal()` (float64). The guard at line 641 is `transaction.Result.Successful()`, so failed transactions retain the string sentinel.
- `internal/transform/operation.go:TransformOperation:30-100` — Confirmed: line 54 calls `extractOperationDetails()`, lines 92/97 assign the result map to both `OperationDetails` and `OperationDetailsJSON` without any type normalization.
- `github.com/stellar/go-stellar-sdk/amount/main.go:134-158` — Confirmed: `amount.String()` returns Go `string` via `big.Rat.FloatString(7)`.
- `internal/utils/main.go:ConvertStroopValueToReal` — Returns Go `float64` via division by 1e7.
- `internal/transform/operation.go:660-699` — Same pattern in `PathPaymentStrictSend`: `details["amount"] = amount.String(0)` (string) at line 672, overwritten to `utils.ConvertStroopValueToReal()` (float64) at line 696 on success.
- `internal/transform/operation.go:1364-1416` — Alternative `transactionOperationWrapper.Details()` method uses `amount.String()` consistently for ALL amounts (no type flip), confirming string-only is the canonical upstream pattern.

### Findings

1. **Primary bug confirmed**: `source_amount` in `path_payment_strict_receive` exports as JSON string `"0.0000000"` for failed transactions and JSON number (e.g., `894.6764349`) for successful transactions. Any downstream consumer (BigQuery, Parquet ingestion, schema-enforcing pipelines) that infers column type from a batch of rows will encounter a type conflict.

2. **Same bug in sibling operation**: `details["amount"]` in `path_payment_strict_send` has the identical string-to-float64 type flip (lines 672 vs 696). The PoC should test both.

3. **Additional type inconsistency**: `destination_min` in StrictSend (line 674) uses `amount.String()` (always string), while `source_max` in StrictReceive (line 633) uses `ConvertStroopValueToReal()` (always float64). These are different keys so no per-key type flip, but the inconsistency across the amount-field family is notable.

4. **Root cause**: `extractOperationDetails()` predominantly uses `ConvertStroopValueToReal()` (float64) for amount fields, but the sentinel/default values for path payment unknowns use `amount.String(0)` (string). The alternative `transactionOperationWrapper.Details()` method (used for effects, not exports) is consistent — it uses `amount.String()` for everything.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go`
- **Setup**: Construct a `path_payment_strict_receive` operation with a failed transaction result (e.g., `xdr.TransactionResultCodeTxFailed`). Use existing test helpers like `utils.CreateSampleTx()` or construct a minimal `ingest.LedgerTransaction` with `Result.Successful()` returning false.
- **Steps**:
  1. Call `TransformOperation()` with the failed `path_payment_strict_receive` operation.
  2. Extract `outputDetails["source_amount"]` from the result.
  3. Check its Go type using a type switch or `reflect.TypeOf()`.
  4. Also test the successful case and verify the type is `float64`.
- **Assertion**: Assert that the Go type of `details["source_amount"]` is the same (either both `string` or both `float64`) regardless of transaction success. Currently, failed returns `string` and successful returns `float64`. Optionally, repeat for `path_payment_strict_send` with `details["amount"]`.
