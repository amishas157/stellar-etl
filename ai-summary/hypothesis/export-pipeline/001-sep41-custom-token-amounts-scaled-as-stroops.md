# H001: SEP-41 custom token amounts are incorrectly scaled as stroops

**Date**: 2026-04-10
**Subsystem**: export-pipeline
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_token_transfers` exports a non-SAC SEP-41 token event, the numeric `amount` should preserve the token-unit quantity emitted by the contract event. If the upstream token-transfer processor surfaces `amount_raw="1000"` for a custom token transfer, the exported `amount` should represent that same token quantity rather than applying Stellar stroop scaling that only makes sense for XLM / classic issued-asset amounts.

## Mechanism

`transformEvents()` parses every token-transfer `AmountRaw` string with `strconv.ParseFloat()` and then unconditionally multiplies it by `1e-7`, regardless of whether the event came from a classic asset or a custom SEP-41 contract. The upstream SDK's custom-token parsers build those event amounts from `amount.String128Raw(amt)`, so a contract-emitted raw quantity like `"1000"` is exported as `0.0001`, understating every custom-token transfer by a factor of 10,000,000 while still looking numerically plausible.

## Trigger

Run `export_token_transfers` on a ledger containing a non-SAC SEP-41 V3 or V4 contract event whose on-chain amount is `i128(1000)`. The SDK will surface `AmountRaw="1000"`, but the ETL will export `amount=0.0001`.

## Target Code

- `internal/transform/token_transfer.go:47-73` — every event arm parses `AmountRaw` and multiplies it by `0.0000001`
- `internal/transform/token_transfer.go:108-126` — the scaled float is written into `TokenTransferOutput.Amount`
- `internal/transform/token_transfer_test.go:52-149` — current expectations only cover classic stroop-scaled cases

## Evidence

The upstream custom-token parser uses `amount.String128Raw(amt)` for SEP-41 V3 and V4 events before constructing `Transfer`, `Mint`, `Burn`, and `Clawback` messages, so the event amount is already a raw token-unit string rather than a stroop value. Its own tests assert that a V4 custom-token transfer with `amount=1000` yields `event.GetTransfer().Amount == "1000"` and `event.GetAsset() == nil`, confirming that these non-SAC events are not normalized to 7-decimal Stellar asset units before they reach this repository.

## Anti-Evidence

For native XLM and classic issued-asset token transfers, the SDK does surface stroop-denominated strings, so multiplying by `1e-7` is correct for those paths. The bug only appears on custom SEP-41 events where the asset payload is absent and the amount string is already the contract's raw token quantity.
