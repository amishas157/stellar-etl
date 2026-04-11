# H001: Token-transfer JSON `amount` rounds large raw i128 values through `float64`

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For every token-transfer row, `amount` should be the exact decimal rendering of `amount_raw` scaled by 1e7, so two distinct on-chain raw amounts always export as two distinct numeric amounts.

## Mechanism

`transformEvents()` converts the raw i128 decimal string into a `float64` with `strconv.ParseFloat`, then multiplies by `0.0000001` before storing it in `TokenTransferOutput.Amount`. Large integers above the 53-bit mantissa limit cannot be represented exactly, so low-order digits are rounded away even though `amount_raw` still carries the exact source value; downstream consumers who read `amount` get a plausible but wrong monetary value.

## Trigger

Export any token transfer whose raw event amount needs more than 53 bits of integer precision, such as `amount_raw = "9007199254740993"` or `amount_raw = "9223372036854775807"`. In Go, those parse as `9007199254740992` and `9223372036854775808` respectively before the 1e7 scale factor is applied.

## Target Code

- `internal/transform/token_transfer.go:44-73` — parses raw event amounts with `strconv.ParseFloat` and rescales them into `float64`
- `internal/transform/token_transfer.go:108-126` — exports the rounded float into `TokenTransferOutput.Amount` while also preserving the exact `AmountRaw`
- `internal/transform/schema.go:659-677` — publishes `amount` as a numeric JSON field that downstream systems can consume directly

## Evidence

The export path keeps the exact source string in `AmountRaw` but still derives `Amount` through a lossy `float64` conversion. A direct Go reproduction shows `strconv.ParseFloat("9007199254740993", 64)` becomes `9007199254740992`, so the exported scaled amount is rounded before JSON encoding ever happens.

## Anti-Evidence

`AmountRaw` remains exact, so consumers that deliberately ignore `amount` can recover the source integer. The bug matters specifically because the ETL also emits a first-class numeric `amount` field that looks authoritative and is easier for analytics pipelines to use.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete token transfer export path from the upstream SDK's `amount.String128Raw()` through `transformEvents()` in `token_transfer.go` to the final `TokenTransferOutput` struct. The SDK extracts i128 values from Soroban contract events and emits them as raw decimal strings (confirmed at `go-stellar-sdk/processors/token_transfer/contract_events.go:82`). The ETL then parses these arbitrary-precision integer strings through `strconv.ParseFloat(amount, 64)` on lines 52/57/62/67/72, losing precision for any value exceeding 2^53 (≈9.007×10^15). The lossy float64 is multiplied by 0.0000001 and stored in `Amount`, while the exact string is preserved in `AmountRaw`.

### Code Paths Examined

- `go-stellar-sdk/processors/token_transfer/contract_events.go:78-82` — Upstream SDK extracts i128 from XDR via `value.GetI128()` and converts to decimal string via `amount.String128Raw(amt)`. This string can represent values up to 2^127-1.
- `internal/transform/token_transfer.go:37-76` — `transformEvents()` switches on event type (Transfer/Mint/Burn/Clawback/Fee). Each arm reads `evt.*.Amount` (a raw i128 decimal string), then calls `strconv.ParseFloat(amount, 64)` discarding the error, and multiplies by 0.0000001. No range check or precision guard exists.
- `internal/transform/token_transfer.go:108-126` — The lossy `amountFloat` is assigned to `TokenTransferOutput.Amount` (float64) while the exact string goes to `AmountRaw`.
- `internal/transform/schema.go:659-677` — `TokenTransferOutput.Amount` is typed `float64` with json tag `"amount"`, exported directly to JSON consumers.
- No Parquet schema exists for `TokenTransferOutput` (grep confirmed no match in `schema_parquet.go`), so this affects the JSON export path only.

### Findings

The bug is confirmed:

1. **Lossy conversion**: `strconv.ParseFloat` converts an arbitrary-precision integer string to float64, which has only 53 bits of mantissa. Any raw amount ≥ 2^53 = 9,007,199,254,740,993 stroops (~900.7 million XLM) loses precision. For Soroban custom tokens with different decimal semantics, the raw i128 value can easily exceed this threshold.

2. **Silent error discarding**: The error return from `strconv.ParseFloat` is discarded with `_` (lines 52, 57, 62, 67, 72). For values that overflow float64 range entirely (theoretically possible with i128), this would silently produce `+Inf` or `0` with no error propagation.

3. **Authoritative field exposure**: The `Amount` field is a first-class JSON member that downstream analytics systems (BigQuery, compliance pipelines) will naturally prefer over the string `AmountRaw` for numeric operations. The field looks authoritative but silently carries rounded values.

4. **Scope of impact**: For classic XLM (7-digit precision), the threshold of ~900 million XLM is high but achievable (SDF manages billions of XLM). For Soroban tokens with non-7-digit decimal schemes, the raw i128 values routinely exceed 2^53, making this a common rather than edge-case bug.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go`
- **Setup**: Construct a `token_transfer.TokenTransferEvent` with a Transfer event whose `Amount` string is `"9007199254740993"` (2^53 + 1). Build minimal `LedgerCloseMeta` with a valid ledger sequence and closed-at time.
- **Steps**: Call `transformEvents()` with the crafted event. Extract the `Amount` and `AmountRaw` fields from the returned `TokenTransferOutput`.
- **Assertion**: Assert that `output.AmountRaw == "9007199254740993"` (exact) but `output.Amount != 900719925.4740993` (the mathematically correct scaled value). Specifically, `output.Amount` will equal `900719925.4740992` due to float64 rounding. This demonstrates that two distinct on-chain amounts (`9007199254740992` and `9007199254740993`) produce identical `Amount` values in the export.
