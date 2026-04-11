# H003: Failed path-payment rows fabricate zero outcome amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a path payment fails, the exported operation details should preserve the request parameters (`send_max`, `dest_min`, requested destination amount) but should not fabricate an executed outcome amount. `path_payment_strict_receive` should leave `source_amount` absent, and `path_payment_strict_send` should leave `amount` absent, because there is no successful result arm providing the realized transfer amount.

## Mechanism

`extractOperationDetails()` seeds `path_payment_strict_receive.source_amount` with `amount.String(0)` and `path_payment_strict_send.amount` with `amount.String(0)`, then overwrites those placeholders only inside the `transaction.Result.Successful()` branch. For failed operations, the overwrite never happens, so the exporter emits a concrete zero-valued outcome that looks like a real executed amount rather than "no result."

## Trigger

Export any failed `PATH_PAYMENT_STRICT_RECEIVE` or `PATH_PAYMENT_STRICT_SEND` operation. The resulting JSON row will include `source_amount = "0"` for strict-receive failures or `amount = "0"` for strict-send failures even though no transfer amount was actually determined.

## Target Code

- `internal/transform/operation.go:619-656` — strict-receive path initializes `source_amount` to zero before checking for success
- `internal/transform/operation.go:660-697` — strict-send path initializes `amount` to zero before checking for success
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/xdr/xdr_generated.go:25898-26046` — the operation XDR contains request bounds (`SendMax`, `DestAmount`, `SendAmount`, `DestMin`), but the realized result amount comes only from the success result arm

## Evidence

The code itself distinguishes request fields from realized result fields by fetching `result.SendAmount()` / `result.DestAmount()` only on success. That makes the zero literal a placeholder rather than a protocol value, and because operation details are a free-form JSON map, the transform could omit the field instead of emitting a fake numeric outcome.

## Anti-Evidence

The operation result code is exported alongside the details map, so careful consumers can infer the payment failed. But the details payload itself is still wrong: it contains a specific executed amount that never occurred, and zero is a plausible value for downstream systems that do not special-case every failure code.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced both `extractOperationDetails()` (line 584) and the `Details()` method on `transactionOperationWrapper` (line 1364) in `internal/transform/operation.go`. Both implement the same pattern: initialize the outcome amount to `amount.String(0)` and overwrite only on `transaction.Result.Successful()`. Then traced the canonical upstream implementation in `stellar/go` SDK at `processors/operation/path_payment_strict_receive_details.go` and `path_payment_strict_send_details.go`. The upstream SDK uses Go struct zero values (int64 `0`) for the same fields when the transaction is not successful — semantically identical behavior.

### Code Paths Examined

- `internal/transform/operation.go:619-656` — `extractOperationDetails` strict-receive arm: sets `source_amount = amount.String(0)`, overwrites with `result.SendAmount()` only on success
- `internal/transform/operation.go:660-697` — `extractOperationDetails` strict-send arm: sets `amount = amount.String(0)`, overwrites with `result.DestAmount()` only on success
- `internal/transform/operation.go:1379-1400` — `Details()` strict-receive arm: identical pattern with `amount.String(0)` default
- `internal/transform/operation.go:1402-1416` — `Details()` strict-send arm: identical pattern with `amount.String(0)` default
- `stellar/go processors/operation/path_payment_strict_receive_details.go` — upstream SDK: `SourceAmount` field defaults to Go zero value `0` when not successful
- `stellar/go processors/operation/path_payment_strict_send_details.go` — upstream SDK: `Amount` field defaults to Go zero value `0` when not successful

### Why It Failed

The behavior is **working as designed** and matches the canonical Horizon SDK implementation exactly. The upstream `stellar/go` SDK (`processors/operation/path_payment_strict_receive_details.go` and `path_payment_strict_send_details.go`) uses the same pattern: the outcome amount field defaults to zero (via Go's int64 zero value) and is only populated from the result when `Transaction.Successful()` is true. The stellar-etl code explicitly mirrors this with `amount.String(0)`. This is the ecosystem-standard convention for representing failed path payment outcomes, not a data fabrication bug. The hypothesis's "Expected Behavior" (omitting the field entirely) does not match the actual Horizon API contract.

### Lesson Learned

Before claiming an ETL emits "fabricated" values, check the canonical upstream implementation (Horizon SDK). If the upstream uses the same zero-default pattern for failed operations, the ETL is correctly replicating the ecosystem convention, not introducing a bug.
