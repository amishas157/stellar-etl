# H001: Claimable-balance amounts round large stroop values through `float64`

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`ClaimableBalanceOutput.AssetAmount` should equal the exact decimal value of the on-chain `ClaimableBalanceEntry.Amount` stroop integer divided by `10^7`. For example, a balance amount of `9007199254740993` stroops should export `asset_amount = 900719925.4740993`.

## Mechanism

`TransformClaimableBalance()` converts the exact `int64` stroop amount by first casting it to `float64` and then dividing by `1.0e7`. Once the stroop value exceeds float64's exact integer range, that cast rounds before scaling, so adjacent valid claimable-balance amounts collapse to the same exported JSON number even though the underlying ledger values differ.

## Trigger

Process any claimable-balance ledger entry whose `ClaimableBalanceEntry.Amount` exceeds float64's exact-integer range, such as `9007199254740993`. The exported row will preserve the correct `balance_id` and asset metadata but emit `asset_amount = 900719925.4740992` instead of `900719925.4740993`.

## Target Code

- `internal/transform/claimable_balance.go:42-67` — direct `float64(outputAmount) / 1.0e7` conversion for `AssetAmount`
- `internal/transform/schema.go:153-169` — claimable-balance schema exposes `asset_amount` as `float64`
- `internal/transform/claimable_balance_test.go:107-131` — tests only cover small values and never exercise large-amount precision boundaries

## Evidence

This path does not reuse the already-known `ConvertStroopValueToReal()` helper; it performs its own float64 cast inline. That makes claimable balances a separate live corruption path: large claimable-balance amounts can be wrong even if downstream reviewers only checked callers of the shared helper.

## Anti-Evidence

Current fixtures use a small amount (`9990000000` stroops), so the issue stays invisible in repository tests. Consumers that recompute from raw XDR could recover, but the exported `asset_amount` field itself is silently wrong in the normal JSON output.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated. Related to `utilities/001-float64-stroop-collisions` (which covers `ConvertStroopValueToReal` callers) but this is a distinct inline code path not enumerated in that finding.

### Trace Summary

`TransformClaimableBalance()` in `claimable_balance.go:24-77` extracts `balanceEntry.Amount` (type `xdr.Int64`, which is Go `int64`) into `outputAmount` at line 48, then at line 66 performs `float64(outputAmount) / 1.0e7`. This is a two-step lossy conversion: the `float64()` cast rounds any int64 exceeding 2^53, and then the division by `1.0e7` introduces a second rounding. The result is stored in `ClaimableBalanceOutput.AssetAmount` (type `float64`, schema.go:160). Unlike `TokenTransferOutput` which preserves the exact raw amount alongside the float, `ClaimableBalanceOutput` has no stroop-exact companion field, making recovery impossible from exported data alone.

### Code Paths Examined

- `internal/transform/claimable_balance.go:48` — `outputAmount := balanceEntry.Amount` assigns `xdr.Int64` (Go `int64`) to local variable
- `internal/transform/claimable_balance.go:66` — `AssetAmount: float64(outputAmount) / 1.0e7` performs inline double-rounding conversion
- `internal/transform/schema.go:153-169` — `ClaimableBalanceOutput.AssetAmount` is `float64` with no parallel exact-integer field
- `internal/transform/claimable_balance_test.go:107-132` — test fixture uses `AssetAmount: 999` (9,990,000,000 stroops), well below the 2^53 threshold
- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal` uses `big.NewRat().Float64()` (single rounding) — claimable balance does NOT use this helper
- `internal/transform/schema_parquet.go:131-134` — `ClaimableBalanceOutputParquet` is commented out/not implemented, so only JSON output is affected

### Findings

1. **Distinct code path**: The claimable balance conversion at line 66 performs `float64(int64) / 1.0e7` — a direct cast plus floating-point division. This is different from `ConvertStroopValueToReal` which uses `big.NewRat(int64, 10000000).Float64()`. The inline path double-rounds (once at cast, once at division) while the helper single-rounds (from exact rational to float64).

2. **Not covered by existing findings**: Published finding `utilities/001-float64-stroop-collisions` explicitly enumerates account, trustline, offer, and trade callers of `ConvertStroopValueToReal`. Claimable balance is absent because it doesn't use the helper. A targeted fix of that helper would leave this path broken.

3. **No recovery path**: Unlike `TokenTransferOutput` which has both `Amount` (float64) and `AmountRaw` (string), `ClaimableBalanceOutput` only has `AssetAmount` (float64). Downstream consumers cannot recover the exact stroop value from the exported JSON.

4. **Threshold**: The bug triggers when `ClaimableBalanceEntry.Amount` exceeds 2^53 = 9,007,199,254,740,992 stroops ≈ 900.7M XLM. While large for a single claimable balance, this is within the valid range of the int64 amount field and possible for large holders or issued assets.

### PoC Guidance

- **Test file**: `internal/transform/claimable_balance_test.go` (append new test)
- **Setup**: Construct a `ClaimableBalanceEntry` with `Amount = xdr.Int64(9007199254740993)` (2^53 + 1) inside a valid `ingest.Change` and `LedgerHeaderHistoryEntry`. Use the existing `makeClaimableBalanceTestInput()` as a template but override the `Amount` field.
- **Steps**: Call `TransformClaimableBalance(change, header)` and extract `output.AssetAmount`.
- **Assertion**: Assert that `output.AssetAmount != 900719925.4740993` (demonstrating precision loss). Also show that `float64(9007199254740993) / 1.0e7` produces `900719925.4740992` — a value that differs from the mathematically correct result. Compare with `big.NewRat(9007199254740993, 10000000).Float64()` to show that even single-rounding gives a different (still imprecise) result, confirming the double-rounding makes it worse.
