# H005: Token-transfer export rounds large raw amounts before scaling

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TokenTransferOutput.Amount` should equal the exact decimal value of `AmountRaw / 10^7` for any valid token-transfer event that the upstream processor already decoded into a raw i128 decimal string. For example, a raw amount of `9007199254740993` should export `amount = 900719925.4740993`.

## Mechanism

`transformEvents()` parses the upstream raw decimal string with `strconv.ParseFloat(..., 64)` and then scales the rounded float by `1e-7`. Once the raw integer exceeds float64's exact-integer range, the conversion rounds before scaling, so the exported `amount` becomes a nearby but wrong decimal value — e.g. `9007199254740993` is rounded to `9007199254740992`, which exports as `900719925.4740992`.

## Trigger

Process any token-transfer event whose raw amount exceeds float64's exact integer range, such as:

1. A SEP-41/custom-token transfer, mint, burn, clawback, or fee event
2. `AmountRaw = "9007199254740993"` (or any larger i128 value with non-zero low bits)

The exported row will keep `amount_raw = "9007199254740993"` but emit `amount = 900719925.4740992`.

## Target Code

- `internal/transform/token_transfer.go:47-73` — every event type parses the raw decimal string with `strconv.ParseFloat(..., 64)` and multiplies by `0.0000001`
- `internal/transform/schema.go:659-677` — schema exposes both `amount` and exact `amount_raw`, so cross-field disagreement becomes observable
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:76-82` — fee events are emitted as raw i128 decimal strings via `amount.String128Raw`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/contract_events.go:244-250` — custom-token events use the same raw i128 string path
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/token_transfer_event.pb.go:123-129` — transfer events carry `Amount` as a string
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/processors/token_transfer/token_transfer_event.pb.go:371-376` — fee events also carry `Amount` as a string

## Evidence

The transform already preserves the exact raw amount string in `AmountRaw`, which makes the corruption directly visible: `Amount` is derived from the same source but via an inexact float64 parse. A concrete float64 check shows `float64("9007199254740993")` becomes `9007199254740992.0` before scaling, so the exported decimal is guaranteed to be off by one stroop-equivalent at the scaled precision.

## Anti-Evidence

Current fixtures use tiny raw values like `"100"`, so the rounding bug stays invisible in existing tests. Consumers that ignore `amount` and recompute from `amount_raw` can recover, but any downstream system that trusts the primary numeric field will ingest a silent financial discrepancy.
