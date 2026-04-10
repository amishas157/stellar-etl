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

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full `transformEvents()` path in `token_transfer.go`. Each event type (Transfer, Mint, Burn, Clawback, Fee) extracts the amount as a string from the SDK event, then calls `strconv.ParseFloat(amount, 64)` with the error discarded, followed by multiplication by `0.0000001`. The result is stored in `TokenTransferOutput.Amount` (float64) alongside `AmountRaw` (string). For stroop values exceeding 2^53 (9,007,199,254,740,992), `ParseFloat` rounds to the nearest representable float64 before the scaling multiplication, introducing a double-rounding path. The rest of the codebase uses `ConvertStroopValueToReal` via `big.NewRat` (single-rounding), but the token transfer code does not.

### Code Paths Examined

- `internal/transform/token_transfer.go:47-76` — Five event-type cases each repeat the same `ParseFloat` + multiply pattern; error from ParseFloat is silently discarded via `_`
- `internal/transform/token_transfer.go:108-126` — `TokenTransferOutput` struct literal assigns `Amount: amountFloat` (lossy float64) and `AmountRaw: amount` (exact string)
- `internal/transform/schema.go:659-677` — `TokenTransferOutput` schema defines `Amount float64` and `AmountRaw string` as sibling fields
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal` uses `big.NewRat(int64(input), 10000000).Float64()` for a single-rounding conversion; token transfer code does NOT use this function

### Findings

1. **Double-rounding path confirmed.** The token transfer code parses a string to float64 (first rounding at 2^53 boundary) then multiplies by 0.0000001 (second rounding). This can yield a different float64 than computing the exact rational division. The rest of the codebase avoids this via `big.NewRat`.

2. **Silent error discarding.** `amountFloat, _ = strconv.ParseFloat(amount, 64)` discards parse errors on all five event type branches. If a Soroban i128 token produces an amount string exceeding float64 range (~1.7e308), the error is silently swallowed and `amountFloat` becomes `+Inf` or `0`.

3. **Schema exposes disagreeing fields.** The export contains both `amount` (potentially rounded float64) and `amount_raw` (exact string). Downstream consumers using the float `amount` for monetary calculations will get wrong values for large token transfers.

4. **Soroban tokens amplify the issue.** Soroban contract tokens use i128 amounts, which can vastly exceed 2^53. The amount string from the SDK can represent values up to 2^127, making the float64 `Amount` field essentially meaningless for large Soroban token amounts.

**Severity downgrade rationale (Critical → Medium):** The presence of `AmountRaw` as an exact string representation significantly mitigates downstream impact — consumers CAN obtain the correct value. The threshold for XLM stroops (>900M XLM) is high for native assets, though Soroban tokens easily exceed it. The issue is a genuine data correctness bug but not silently undetectable since the exact value is always co-exported.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go` (or create if absent)
- **Setup**: Construct a mock `TokenTransferEvent` with `Amount = "9007199254740993"` (2^53 + 1 stroops). Use the existing test utility patterns for building mock `LedgerCloseMeta`.
- **Steps**: Call `transformEvents()` with the crafted event. Extract the `Amount` and `AmountRaw` fields from the output.
- **Assertion**: Assert that `Amount` does NOT equal `AmountRaw` parsed to float64 and divided by 1e7 with exact arithmetic. Specifically, show that `Amount == 900719925.4740992` (rounded) while `AmountRaw == "9007199254740993"` (exact), demonstrating the disagreement. Additionally test with a value like `"18014398509481985"` (2^54 + 1) to show larger errors.
