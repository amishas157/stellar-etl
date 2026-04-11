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
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:parseCustomTokenEventV3:244-250` — upstream parser reads token amounts from `ScVal::I128`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:parseFeeEvents:76-82` — fee events also serialize raw amounts with `amount.String128Raw`

## Evidence

The upstream processor hands stellar-etl a decimal string specifically so the full i128 amount survives protocol parsing. The ETL immediately collapses that string into `float64`, even though `TokenTransferOutput` also keeps the exact `AmountRaw` side-by-side. Because the parse error is ignored, overflow does not even surface as a failed export; it just changes the numeric value in-place.

## Anti-Evidence

Small token amounts, including the checked-in test fixture value `"100"`, survive this path with only ordinary floating-point noise. Downstream consumers can sometimes recover the correct value from `amount_raw`, but the exported `amount` column itself is still wrong and silently wrong.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete path from upstream SDK through stellar-etl transform. The upstream `contract_events.go` extracts i128 values from XDR via `value.GetI128()` and converts them to full-precision decimal strings via `amount.String128Raw(amt)` (which uses `big.Int` arithmetic). These strings flow through protobuf event constructors (`NewTransferEvent`, `NewMintEvent`, etc.) as the `Amount string` field. In `token_transfer.go:transformEvents()`, every event arm calls `strconv.ParseFloat(amount, 64)` with the error discarded via `_`, then multiplies by `0.0000001`. The resulting `float64` is written to `TokenTransferOutput.Amount`. For raw amounts exceeding 2^53 (9,007,199,254,740,992), `ParseFloat` silently rounds to the nearest representable float64, producing an incorrect `Amount` value while `AmountRaw` preserves the exact string.

### Code Paths Examined

- `internal/transform/token_transfer.go:37-130` — `transformEvents()` switch over event types; lines 52, 57, 62, 67, 72 each call `strconv.ParseFloat(amount, 64)` with error discarded via `_`; line 120 assigns the lossy `amountFloat` to `Amount`; line 119 assigns the exact string to `AmountRaw`
- `internal/transform/schema.go:659-677` — `TokenTransferOutput` struct: `Amount float64` at line 670, `AmountRaw string` at line 671
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/amount/main.go:166-173` — `String128Raw()` uses `big.Int` to produce exact decimal representation of i128; no truncation
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/processors/token_transfer/contract_events.go:78-82,245-250` — upstream extracts i128 via `GetI128()` and converts to string via `String128Raw()`; the raw string is passed directly to event constructors as the `Amount` field

### Findings

**Rounding confirmed**: `float64` has 53 bits of significand, giving exact integer representation only up to 2^53 = 9,007,199,254,740,992. Python verification confirms `float64(9007199254740993)` rounds to `9007199254740992`. After division by 1e7, this produces `900719925.4740992` instead of the correct `900719925.4740993`. The discrepancy grows with magnitude.

**+Inf claim is INCORRECT**: The maximum i128 value (~1.7×10^38) is well within `float64` range (~1.8×10^308). `strconv.ParseFloat` will never return +Inf for any valid i128 value. It will round, but not overflow. The hypothesis overstates this aspect.

**Custom tokens are the high-risk trigger**: SAC tokens use 7-decimal stroop precision, so the rounding threshold is ~900 million XLM — large but reachable. Custom Soroban tokens have arbitrary precision; a token using 18 decimal places (like Ethereum's wei) would hit the rounding boundary at just ~9 tokens (10^18 raw > 2^53). This makes the bug easily triggerable for custom token ecosystems.

**Error discarding is real but misleading**: The `_` on `ParseFloat` discards the error, but `ParseFloat` does not return an error for large-but-finite values — it silently rounds and returns `nil`. The error would only be non-nil for actual overflow to ±Inf (impossible for i128) or malformed strings. So the discarded error is a code smell but not the primary mechanism of data loss.

**No guards exist**: There is no validation anywhere in the path that checks whether the `float64` conversion was lossless. The upstream SDK deliberately uses `big.Int` and string serialization to preserve full i128 precision, but the ETL immediately discards that precision.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go`
- **Setup**: Construct a mock `TokenTransferEvent` with `Transfer.Amount` set to `"9007199254740993"` (2^53 + 1). Use existing test helpers to build a minimal `LedgerCloseMeta`.
- **Steps**: Call `transformEvents()` with the mock event. Extract the `Amount` field from the returned `TokenTransferOutput`.
- **Assertion**: Assert that `output.Amount` does NOT equal `900719925.4740993` (the mathematically correct value). Specifically, assert `output.Amount == 900719925.4740992` to demonstrate the rounding. Also assert `output.AmountRaw == "9007199254740993"` to show the raw value is preserved but the float is wrong.
- **Stretch**: Also test with `"18014398509481985"` (2^54 + 1) to show the error magnitude doubles with each power of 2.
