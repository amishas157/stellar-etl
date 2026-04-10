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

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete data path from the SDK's `parseCustomTokenEventV3`/`parseCustomTokenEventV4` through `parseEvent` to the ETL's `transformEvents`. The SDK uses `amount.String128Raw(amt)` (which returns the raw i128 integer as a decimal string with NO stroop conversion) for ALL token events, then sets `Asset = nil` for custom SEP-41 tokens (no SAC validation match). The ETL's `transformEvents` unconditionally multiplies every parsed amount by `0.0000001` regardless of token type. For SAC tokens this is correct (they ARE in stroops), but for custom SEP-41 tokens the raw contract amount is divided by 10^7, producing silently wrong `Amount` values.

### Code Paths Examined

- `go-stellar-sdk/amount/main.go:166-173` — `String128Raw` returns raw `big.Int.String()` with NO stroop conversion, confirming the amount string is the unscaled i128 value
- `go-stellar-sdk/processors/token_transfer/contract_events.go:238-321` — `parseCustomTokenEventV3` calls `amount.String128Raw(amt)` at line 250, passes result as amount to `NewTransferEvent/NewMintEvent/NewBurnEvent/NewClawbackEvent` with `nil` asset
- `go-stellar-sdk/processors/token_transfer/contract_events.go:324-360` — `parseCustomTokenEventV4` same pattern at line 357, also passes `nil` asset
- `go-stellar-sdk/processors/token_transfer/contract_events.go:110-200` — `parseEvent` attempts SAC validation; for custom tokens, contract ID won't match expected asset contract ID, so `SetAsset` is never called, leaving Asset as `nil`
- `internal/transform/token_transfer.go:47-73` — `transformEvents` switch over event types: all five arms (Transfer, Mint, Burn, Clawback, Fee) unconditionally do `amountFloat = amountFloat * 0.0000001`
- `internal/transform/token_transfer.go:132-151` — `getAssetFromEvent` returns empty strings when `event.GetAsset()` is nil (custom token), proving custom tokens DO pass through with empty asset fields but still get the wrong Amount
- `internal/transform/token_transfer_test.go:156-278` — test inputs all use `Asset_IssuedAsset` (SAC pattern); no custom token test case exists

### Findings

The bug is confirmed. The `Amount` field in `TokenTransferOutput` will be incorrect by a factor of 10^7 for every custom SEP-41 token event. A custom contract transfer of `i128(1000)` produces `AmountRaw="1000"` and `Amount=0.0001` instead of the intended value. The `AmountRaw` field is unaffected (it preserves the raw string), but the `Amount` float64 field — the primary numeric value for downstream analytics — is silently corrupted.

The ETL already has the information needed to distinguish SAC from custom tokens: `event.GetAsset()` returns `nil` for custom tokens. The `getAssetFromEvent` function uses this to populate asset fields, but the amount scaling path doesn't check it.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go`
- **Setup**: Add a new test event with `Asset: nil` (no `Asset_IssuedAsset` or `Asset_Native` set) to simulate a custom SEP-41 token transfer. Set `Amount: "1000"` in the Transfer struct.
- **Steps**: Call `transformEvents` with the custom-token event and a valid `LedgerCloseMeta`.
- **Assertion**: Assert that the output `Amount` field equals `0.0001` (demonstrating the incorrect stroop scaling). The correct behavior would be to NOT scale the amount when the asset is nil (custom token), yielding `Amount=1000.0`. Additionally, verify that `AmountRaw` correctly equals `"1000"` and `AssetType` is empty string.
