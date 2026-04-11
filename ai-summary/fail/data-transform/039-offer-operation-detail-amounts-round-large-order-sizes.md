# H004: Offer-operation detail amounts round large order sizes

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For successful `manage_buy_offer`, `manage_sell_offer`, and `create_passive_sell_offer` operations, `history_operations.details.amount` should preserve the exact order size implied by the XDR. Two offers that differ by 1 stroop in `BuyAmount` or `Amount` should not export the same JSON value.

## Mechanism

`extractOperationDetails()` converts offer amounts through `utils.ConvertStroopValueToReal()` before placing them into the details map. In the same branches, the exporter already keeps an exact rational `price_r`, and the sibling wrapper emits `amount.String(...)` for the order size, so large offer amounts can be silently rounded even though the details payload already supports exact string/rational companions.

## Trigger

Process any successful offer operation with a very large amount, for example:

1. `ManageBuyOfferOp.BuyAmount = 90071992547409931`
2. `ManageSellOfferOp.Amount = 90071992547409931`
3. `CreatePassiveSellOfferOp.Amount = 90071992547409931`

Compare the exported `details.amount` against `amount.String(...)` or against an adjacent 1-stroop amount; distinct order sizes will collapse to the same floating-point export.

## Target Code

- `internal/transform/operation.go:701-755` — offer-operation branches convert `amount` through `utils.ConvertStroopValueToReal()`
- `internal/transform/operation.go:1424-1455` — sibling wrapper emits exact `amount.String(...)` values for the same operation families
- `internal/transform/operation.go:409-420` — these branches already preserve exact `price_r`, showing the details payload is not limited to bare floats
- `internal/transform/schema.go:136-150` — operation details are a flexible map rather than a typed float64 schema

## Evidence

The code already treats price precision as important enough to export both rounded `price` and exact `price_r`; amount precision is therefore an obvious outlier in the same offer-detail payload. The wrapper-side exact-string implementation shows that the lossy amount conversion is a local choice, not a protocol limitation.

## Anti-Evidence

Existing tests cover only modest offer sizes, so the rounded and exact encodings coincide in the current suite. A reviewer may argue that `amount` has historically been numeric in operation details, but that does not explain why the same branch keeps exact price metadata while discarding exact size metadata.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (but substantially equivalent to rejected pattern 034-038)
**Failed At**: reviewer

### Trace Summary

The offer operation branches at `extractOperationDetails()` lines 701-755 convert amounts via `utils.ConvertStroopValueToReal()`, which uses `big.NewRat(int64(input), int64(10000000)).Float64()` — the best possible float64 conversion available in Go. The hypothesis cites `transactionOperationWrapper.Details()` (line 1364) as evidence of a precision-preserving alternative, but `.Details()` is never called anywhere in the codebase (confirmed via grep) — it is dead code.

### Code Paths Examined

- `internal/transform/operation.go:701-755` — ManageBuyOffer, ManageSellOffer, CreatePassiveSellOffer branches all use `utils.ConvertStroopValueToReal()` to convert offer amounts to float64
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal()` uses `big.NewRat(int64, 1e7).Float64()`, the highest-precision float64 conversion path available
- `internal/transform/operation.go:1364` — `transactionOperationWrapper.Details()` is defined but never called by any code in the repository (dead code)
- `internal/transform/operation.go:54` — `TransformOperation()` calls `extractOperationDetails()`, not `.Details()`

### Why It Failed

This hypothesis falls into the same category as previously rejected fail entries 034-038: the code uses `ConvertStroopValueToReal()` → `big.NewRat().Float64()`, which is the best possible float64 conversion. The precision loss is an inherent property of the float64 schema type, not a coding error. Unlike VIABLE finding 016 (inferior inline `float64(x)/1e7` conversion) or VIABLE finding 017 (exact string computed then discarded via `ParseFloat`), the offer branches do not bypass a better conversion path and do not compute an exact value only to throw it away. The cited alternative (`transactionOperationWrapper.Details()`) is dead code that is never invoked.

### Lesson Learned

Dead code (`transactionOperationWrapper.Details()`) cannot serve as evidence that a better precision path exists but was not used. Float64 precision loss through `ConvertStroopValueToReal()` / `big.NewRat().Float64()` is an inherent schema design limitation, not a fixable conversion bug. This is the same pattern as fail entries 034-038.
