# H002: Token-transfer `amount` rounds or overflows legitimate i128 values

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TokenTransferOutput.Amount` should preserve the exact 7-decimal token amount represented by `amount_raw`, or fail the transform if the value cannot be represented faithfully. A valid SEP-41 / SAC event with raw amount `9007199254740993` should export `amount = 900719925.4740993`, not a rounded neighbor, and very large but valid i128 token amounts should never silently become `+Inf`.

## Mechanism

`transformEvents()` parses every raw token amount string with `strconv.ParseFloat(..., 64)`, ignores the returned error, and multiplies the result by `1e-7`. Upstream token-transfer events are emitted from Soroban `i128` values via `amount.String128Raw`, so legitimate raw amounts can exceed the exact integer range of `float64` or even overflow it entirely. The transform therefore silently rounds large token amounts and can export `+Inf` for sufficiently large i128 values while still preserving the exact integer in `amount_raw`.

## Trigger

Process any token transfer, mint, burn, clawback, or fee event whose raw amount is outside the exact `float64` integer range, such as:

1. `amount_raw = "9007199254740993"` — should be `900719925.4740993`, but rounds to approximately `900719925.4740992`
2. A much larger valid Soroban i128 amount such as `170141183460469231731687303715884105727` — `ParseFloat` overflows and the export can become `+Inf`

## Target Code

- `internal/transform/token_transfer.go:transformEvents:47-73` — every event arm uses `strconv.ParseFloat(amount, 64)` and discards parse/overflow errors
- `internal/transform/token_transfer.go:transformEvents:108-126` — writes both lossy `Amount float64` and exact `AmountRaw string` to the output row
- `internal/transform/schema.go:TokenTransferOutput:659-677` — schema exposes the corrupted `amount` column as the main numeric field
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:parseCustomTokenEventV3:244-250` — upstream parser reads token amounts from `ScVal::I128`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:parseFeeEvents:76-82` — fee events also serialize raw amounts with `amount.String128Raw`

## Evidence

The upstream processor hands stellar-etl a decimal string specifically so the full i128 amount survives protocol parsing. The ETL immediately collapses that string into `float64`, even though `TokenTransferOutput` also keeps the exact `AmountRaw` side-by-side. Because the parse error is ignored, overflow does not even surface as a failed export; it just changes the numeric value in-place.

## Anti-Evidence

Small token amounts, including the checked-in test fixture value `"100"`, survive this path with only ordinary floating-point noise. Downstream consumers can sometimes recover the correct value from `amount_raw`, but the exported `amount` column itself is still wrong and silently wrong.
