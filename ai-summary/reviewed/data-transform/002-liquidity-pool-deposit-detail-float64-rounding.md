# H002: Liquidity-pool deposit details round large reserve deltas and share amounts

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a successful `liquidity_pool_deposit` operation, the exported detail fields `reserve_a_deposit_amount`, `reserve_b_deposit_amount`, and `shares_received` should preserve the exact decimal values implied by the underlying `xdr.Int64` reserve/share deltas. A one-stroop difference in the pool deltas should remain visible in the operation JSON.

## Mechanism

The deposit detail builder first formats the exact `xdr.Int64` amounts as decimal strings with `amount.String(...)`, then immediately parses those strings back into `float64` with `strconv.ParseFloat(..., 64)`. For large deltas, the float64 parse rounds away low-order digits, so operation details publish plausible-but-wrong deposit and share values even though the preceding string conversion had the exact decimal representation available.

## Trigger

Export a successful `liquidity_pool_deposit` whose reserve delta or `TotalPoolShares` delta exceeds float64's exact-integer range after 7-decimal scaling, such as a deposit delta of `9007199254740993` stroops. The detail map will report `reserve_*_deposit_amount` or `shares_received` as the rounded neighbor value instead of the exact amount.

## Target Code

- `internal/transform/operation.go:974-1019` — successful deposit path computes deltas, then parses `amount.String(...)` back into `float64`
- `internal/transform/operation.go:238-285` — `getLiquidityPoolAndProductDelta()` returns exact `xdr.Int64` reserve/share deltas before the rounding step
- `internal/transform/operation_test.go:1751-1804` — tests cover only tiny deposit/share values

## Evidence

The code already has the exact decimal strings in hand; the only reason precision is lost is the final `ParseFloat` step. This is a live transform-layer bug in the JSON operation export path, not a Parquet-only issue, and it affects three separate monetary fields on the same operation row.

## Anti-Evidence

Typical test fixtures use tiny reserve deltas, so the exported values look correct in current coverage. If downstream consumers ignore these float fields and rebuild amounts from raw XDR elsewhere, they can recover, but the exported operation details themselves are still corrupted.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `TransformOperation` → `extractOperationDetails` → `LiquidityPoolDeposit` case (lines 957-1019). Confirmed that `getLiquidityPoolAndProductDelta` returns exact `xdr.Int64` deltas, `amount.String()` converts them to exact 7-decimal strings via `big.Rat.FloatString(7)`, and then `strconv.ParseFloat(..., 64)` immediately parses back to float64, discarding precision for values beyond 2^53. Three financial fields (`reserve_a_deposit_amount`, `reserve_b_deposit_amount`, `shares_received`) are affected. Notably, the sibling `transactionOperationWrapper.Details()` method (line 1603-1649) stores the same amounts as exact strings, confirming the string representation was available and the ParseFloat step is gratuitous.

### Code Paths Examined

- `internal/transform/operation.go:TransformOperation:30-101` — confirmed it calls `extractOperationDetails` at line 54, producing the exported `OperationDetails`
- `internal/transform/operation.go:extractOperationDetails:957-1019` — confirmed the `LiquidityPoolDeposit` case uses `strconv.ParseFloat(amount.String(depositedA), 64)` on lines 991, 1002, 1015 for all three financial fields
- `internal/transform/operation.go:getLiquidityPoolAndProductDelta:238-285` — confirmed it returns exact `xdr.Int64` deltas computed as `postA - preA`
- `github.com/stellar/go-stellar-sdk/amount/main.go:String:134-136` → `StringFromInt64:155-159` — confirmed `amount.String()` uses `big.Rat.FloatString(7)` which produces an exact decimal string
- `internal/transform/operation.go:transactionOperationWrapper.Details:1603-1649` — confirmed the alternative code path stores the same amounts as strings via `amount.String()` without ParseFloat roundtrip
- `internal/utils/main.go:ConvertStroopValueToReal` — confirmed line 990 uses this for `reserve_a_max_amount` while lines 991-995 use the ParseFloat pattern for `reserve_a_deposit_amount`, showing inconsistent conversion methods for adjacent financial fields

### Findings

1. **Confirmed mechanism**: `amount.String(depositedA)` produces the exact string `"900719925.4740993"` for 9007199254740993 stroops. `strconv.ParseFloat("900719925.4740993", 64)` rounds this to `900719925.4740992` because float64 cannot represent 16 significant digits exactly. The rounded value is stored in the detail map.

2. **Three affected fields**: `reserve_a_deposit_amount` (line 995), `reserve_b_deposit_amount` (line 1006), `shares_received` (line 1019) all use the identical lossy pattern.

3. **Inconsistency with sibling code path**: The `transactionOperationWrapper.Details()` method (lines 1645-1649) stores the same data as exact strings. This proves the exact representation was architecturally available.

4. **Inconsistency within same function**: `reserve_a_max_amount` (line 990) uses `ConvertStroopValueToReal()` while `reserve_a_deposit_amount` (line 991-995) uses `ParseFloat(amount.String())`. Both produce float64 but via different paths.

5. **Not a duplicate**: Published finding `utilities/001-float64-stroop-collisions` covers `ConvertStroopValueToReal` callers (account, trustline, offer, trade). Published finding `data-integrity/002-token-transfer-float64-rounding` covers `transformEvents` in token_transfer.go. This hypothesis targets a distinct code path (`extractOperationDetails` lines 991-1019) using a distinct mechanism (`ParseFloat(amount.String())`) affecting distinct output fields.

### PoC Guidance

- **Test file**: `internal/transform/operation_test.go` (append to existing LP deposit tests around line 1751)
- **Setup**: Construct an `ingest.LedgerTransaction` with a successful `LiquidityPoolDeposit` operation where the post-deposit reserve delta is `9007199254740993` stroops (2^53 + 1). Use `getLiquidityPoolAndProductDelta`-compatible ledger changes with pre/post `LiquidityPoolEntry` values differing by that delta.
- **Steps**: Call `extractOperationDetails()` (or `TransformOperation()`) with the constructed transaction and operation.
- **Assertion**: Assert that `details["reserve_a_deposit_amount"]` equals the exact float64 representation of `900719925.4740993`. The test should fail because the actual value will be `900719925.4740992` (or its float64 neighbor), proving the one-stroop precision loss. Alternatively, construct two transactions differing by 1 stroop in the deposit delta and assert their exported `reserve_a_deposit_amount` values differ.
