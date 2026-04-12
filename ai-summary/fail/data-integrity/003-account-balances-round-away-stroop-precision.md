# H002: Account balances lose exact stroop precision above large native balances

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformAccount()` should export `balance`, `buying_liabilities`, and `selling_liabilities` with exact 7-decimal stroop precision. For example, an account balance of `99_999_999_999_999_999` stroops should serialize as `9999999999.9999999`, not as the rounded whole number `10000000000`.

## Mechanism

The account transformer converts `xdr.Int64` stroop counts to `float64` through `utils.ConvertStroopValueToReal()`, and both JSON and Parquet schemas store those values as `float64` / `DOUBLE`. At large magnitudes, IEEE-754 cannot represent every 1-stroop step, so distinct on-chain balances collapse to the same exported decimal; a Go `json.Marshal` of `ConvertStroopValueToReal(99_999_999_999_999_999)` already emits `10000000000`.

## Trigger

1. Export an account ledger entry whose balance or liabilities exceed the float64 exact-stroop range, e.g. `99_999_999_999_999_999` stroops.
2. Run `export_ledger_entry_changes --export-accounts`.
3. Compare the JSON or Parquet row to the source ledger entry: the exported decimal rounds away the final stroops instead of preserving `...9999999`.

## Target Code

- `internal/utils/main.go:84-87` — converts stroops to `float64`
- `internal/transform/account.go:86-110` — uses that helper for `Balance`, `BuyingLiabilities`, and `SellingLiabilities`
- `internal/transform/schema.go:97-120` — JSON schema stores the monetary fields as `float64`
- `internal/transform/parquet_converter.go:105-130` — Parquet conversion forwards the rounded floats unchanged

## Evidence

The upstream account entry balance and liabilities are integer stroop counts, but the exported schema discards that exact representation entirely. The helper uses `big.Rat(...).Float64()`, so the final narrowing to `float64` is explicit; with a concrete input such as `99_999_999_999_999_999` stroops, Go rounds the exported value to `10000000000`, proving the last 7-decimal digits are not preserved. Unlike token transfers, this table has no parallel raw-amount field to recover the exact on-chain value.

## Anti-Evidence

Small balances remain exact, so existing tests with low-value fixtures will continue to pass and can hide the defect. The negative-value guards in `TransformAccount()` do prevent nonsensical outputs, but they do nothing to protect large positive balances from precision loss.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `success/utilities/001-float64-stroop-collisions.md.gh-published`
**Failed At**: reviewer

### Trace Summary

Traced `ConvertStroopValueToReal` in `internal/utils/main.go:84-87` through `TransformAccount` in `internal/transform/account.go:86-90`, confirming the float64 precision loss mechanism is real. However, this exact mechanism — including account balances and liabilities as explicitly listed affected code — was already confirmed, published, and documented as `success/utilities/001-float64-stroop-collisions.md.gh-published`. Additionally, per-entity variants targeting the same `ConvertStroopValueToReal()` + `big.NewRat().Float64()` path were rejected in bulk as `fail/data-transform/034-040.md`.

### Code Paths Examined

- `internal/utils/main.go:84-87` — `ConvertStroopValueToReal` uses `big.NewRat(int64(input), 10000000).Float64()` with exactness discarded
- `internal/transform/account.go:86-90` — `Balance`, `BuyingLiabilities`, `SellingLiabilities` all use the helper
- `internal/transform/schema.go:97-121` — `AccountOutput` stores these as `float64`
- `internal/transform/schema_parquet.go:78-81` — `AccountOutputParquet` stores them as `DOUBLE`

### Why It Failed

This is an exact duplicate of the already confirmed and published finding `success/utilities/001-float64-stroop-collisions.md.gh-published`, which:
1. Identifies the same root cause: `ConvertStroopValueToReal` in `internal/utils/main.go:84-87`
2. Explicitly lists `internal/transform/account.go:86-90` as affected code
3. Covers account balances and liabilities by name
4. Has been published with Critical severity

Furthermore, per-entity re-submissions of this same hypothesis (including account balance/liabilities specifically) were already investigated and rejected as `fail/data-transform/034-040.md`, with the documented lesson: "code that goes directly from XDR Int64 to float64 via `big.NewRat().Float64()` → NOT_VIABLE schema design limit."

### Lesson Learned

Before submitting entity-specific variants of a precision-loss hypothesis, check `success/utilities/` for cross-cutting findings that already cover the shared helper function and explicitly list the affected entities. The `ConvertStroopValueToReal` float64 limitation has been confirmed, published, and documented as covering all callers including accounts, trustlines, offers, and trades.
