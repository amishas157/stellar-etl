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
