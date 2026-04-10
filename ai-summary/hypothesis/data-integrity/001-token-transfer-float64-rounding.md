# H001: Token transfer `amount` loses stroop precision on large values

**Date**: 2026-04-10
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TokenTransferOutput.Amount` should equal `AmountRaw / 10_000_000` exactly for every legitimate token transfer event. For example, an event with `amount_raw = "9007199254740993"` should export `amount = 900719925.4740993`, not a rounded neighbor.

## Mechanism

`transformEvents()` parses the on-chain decimal string into `float64` and then scales it by `1e-7`. Once the raw stroop string exceeds float64's integer precision, the parse rounds before scaling, so the exported `amount` silently differs from `amount_raw` by one or more stroops.

## Trigger

Process any transfer, mint, burn, clawback, or fee event whose `Amount` string is above the exact-integer range of float64, such as `9007199254740993`.

## Target Code

- `internal/transform/token_transfer.go:44-73` — parses the string amount into `float64`
- `internal/transform/token_transfer.go:108-126` — writes the rounded value into `TokenTransferOutput.Amount`
- `internal/transform/schema.go:659-676` — exposes both `amount` and exact `amount_raw`

## Evidence

The SDK token-transfer event types expose `Amount` as a string, but the transformer immediately calls `strconv.ParseFloat(amount, 64)` and ignores the parse error. The code also preserves the original string as `AmountRaw`, which means the export can contain two disagreeing representations of the same token amount.

## Anti-Evidence

Small amounts still round back to the expected 7-decimal value, so ordinary tests with short numbers will pass. Consumers that ignore `amount` and recompute from `amount_raw` will avoid the corruption.
