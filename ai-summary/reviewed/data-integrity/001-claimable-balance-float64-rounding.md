# H001: Claimable-balance export rounds large stroop amounts before scaling

**Date**: 2026-04-11
**Subsystem**: data-integrity
**Severity**: Critical
**Impact**: Financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`asset_amount` should equal the exact on-chain `ClaimableBalanceEntry.Amount / 10_000_000` for every exported claimable balance. For example, an on-chain amount of `9007199254740993` stroops should export as `900719925.4740993`, not the neighboring double-precision value.

## Mechanism

`TransformClaimableBalance()` converts the integer XDR amount with `float64(outputAmount) / 1.0e7`, which rounds the 64-bit stroop value to `float64` *before* scaling it into display units. Once the balance exceeds the exact integer range of IEEE-754 (`2^53`), the exported JSON number silently shifts by one or more stroops, so downstream balance and entitlement analytics consume a plausible-looking but wrong amount.

## Trigger

Run `export_ledger_entry_changes` over a ledger containing a claimable balance whose `ClaimableBalanceEntry.Amount` is above `9007199254740992` stroops; for instance, `9007199254740993` will export as `900719925.4740992` instead of `900719925.4740993`.

## Target Code

- `internal/transform/claimable_balance.go:47-67` — casts the raw `xdr.Int64` amount to `float64` before dividing by `1e7`
- `internal/transform/schema.go:152-169` — declares `asset_amount` as the exported monetary field for claimable balances
- `cmd/export_ledger_entry_changes.go:160-167` — routes claimable-balance ledger entries through `TransformClaimableBalance()`

## Evidence

Unlike the contract-data and token-transfer paths, claimable balances have no raw integer companion column. The transform uses direct `float64(outputAmount)` conversion instead of an exact decimal/string representation, so the only exported amount field already contains the rounded value.

## Anti-Evidence

Balances at or below `2^53` stroops still convert exactly, so ordinary test fixtures with small amounts will not expose the corruption. The Parquet path is skipped for claimable balances today, so the issue currently manifests in the JSON export path.

---

## Review

**Verdict**: VIABLE
**Severity**: Critical
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the complete conversion path from XDR ingestion through `TransformClaimableBalance()` to JSON export. Confirmed that `claimable_balance.go:66` uses `float64(outputAmount) / 1.0e7` which is a double-rounding path (int64→float64 rounds first, then division rounds again). All other entity transforms in the codebase (account, trustline, offer, trade, liquidity pool, operations) use `utils.ConvertStroopValueToReal()` which performs `big.NewRat(int64(input), 10000000).Float64()` — exact rational division followed by a single float64 rounding. Claimable balance is the sole outlier that bypasses this utility. Furthermore, unlike `TokenTransferOutput` which has a companion `AmountRaw` string field, `ClaimableBalanceOutput` has no lossless fallback — `AssetAmount` is the only exported amount field.

### Code Paths Examined

- `internal/transform/claimable_balance.go:48-66` — `outputAmount` is `balanceEntry.Amount` (type `xdr.Int64`), converted via `float64(outputAmount) / 1.0e7` on line 66. This first casts int64 to float64 (lossy above 2^53), then divides.
- `internal/utils/main.go:85-88` — `ConvertStroopValueToReal()` uses `big.NewRat(int64(input), int64(10000000)).Float64()`, performing exact rational division then a single float64 conversion. This is the correct pattern used by all other entity transforms.
- `internal/transform/schema.go:153-169` — `ClaimableBalanceOutput` declares `AssetAmount float64` with no companion raw integer or string field.
- `internal/transform/account.go:88-90` — Account uses `ConvertStroopValueToReal()` for Balance, BuyingLiabilities, SellingLiabilities.
- `internal/transform/offer.go:90` — Offer uses `ConvertStroopValueToReal()` for Amount.
- `internal/transform/trustline.go:75-79` — Trustline uses `ConvertStroopValueToReal()` for Balance and liabilities.
- `internal/transform/trade.go:139-145` — Trade uses `ConvertStroopValueToReal()` for SellingAmount and BuyingAmount.

### Findings

1. **Double-rounding confirmed**: `float64(9007199254740993)` evaluates to `9007199254740992.0` (losing the +1), then dividing by `1e7` produces `900719925.4740992`. In contrast, `big.NewRat(9007199254740993, 10000000).Float64()` produces `900719925.4740993` — the correctly-rounded nearest float64 to the exact rational value.

2. **Outlier pattern confirmed**: Claimable balance is the only entity type that uses inline `float64(x)/1e7` instead of `ConvertStroopValueToReal()`. This matches Investigation Pattern 5 (outlier detection among sibling functions).

3. **No lossless fallback**: `ClaimableBalanceOutput` has no `amount_raw` string field. The lossy `AssetAmount` float64 is the sole exported representation of the on-chain balance amount. Downstream consumers (BigQuery analytics, compliance reporting) have no way to recover the exact value.

4. **Existing test uses small value**: The test fixture at `claimable_balance_test.go:86` uses `Amount: 9990000000` (999 XLM), well below the 2^53 threshold, so the bug is never exercised in tests.

### PoC Guidance

- **Test file**: `internal/transform/claimable_balance_test.go`
- **Setup**: Construct a `ClaimableBalanceEntry` with `Amount` set to `xdr.Int64(9007199254740993)` (2^53 + 1 stroops). Build an `ingest.Change` and `LedgerHeaderHistoryEntry` using the existing test helpers as a template.
- **Steps**: Call `TransformClaimableBalance(change, header)` and examine `output.AssetAmount`.
- **Assertion**: Assert that `output.AssetAmount` does NOT equal `utils.ConvertStroopValueToReal(xdr.Int64(9007199254740993))`. The production code produces `900719925.4740992` while the correct single-rounding path produces `900719925.4740993`. Also verify via `json.Marshal` that the serialized JSON `asset_amount` field contains the wrong value. Optionally, confirm round-trip: `int64(output.AssetAmount * 1e7)` should NOT equal the original `9007199254740993`.
