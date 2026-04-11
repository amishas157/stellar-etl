# H004: Effect Export Limit Counts Transactions Instead of Effect Rows

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_effects --limit N` should emit at most `N` effect rows, matching the CLI flag text that describes the limit as the maximum number of effects to export. A transaction that expands into multiple effect rows should therefore be truncated or followed by an exact row-level stop once the requested number of effects has been written.

## Mechanism

The command passes the effect limit directly into `GetTransactions()`, which enforces the bound in units of transactions, not effects. `TransformEffect()` then expands each returned transaction into a variable-length slice of `EffectOutput` rows, so `--limit 1` can still write multiple debit/credit/trustline/etc. effects from the first transaction in the range while reporting a normal-looking successful export.

## Trigger

Run `stellar-etl export_effects --limit 1 --start-ledger <S> --end-ledger <E>` on any range whose first returned transaction produces multiple effects (for example, a standard successful payment that yields both debited and credited effects). The correct output should contain exactly one effect row, but the current implementation writes all effects derived from that first transaction.

## Target Code

- `cmd/export_effects.go:21-57` — reads `limit` as "effects" but applies it only when fetching transactions
- `internal/input/transactions.go:GetTransactions:23-70` — stops after `N` transactions, not after `N` effect rows
- `internal/transform/effects.go:23-50` — expands one transaction into many effects
- `internal/utils/main.go:AddArchiveFlags:248-255` — defines the user-visible flag text as "Maximum number of effects to export"

## Evidence

`GetTransactions()` has no visibility into how many `EffectOutput` rows a transaction will later generate, yet `export_effects` never applies a second limit after `TransformEffect()`. The command comment also documents the flag as an effect count, not a transaction count, so the current control flow silently violates the exported row bound instead of merely using a confusing internal unit.

## Anti-Evidence

If the first `N` transactions each happen to generate exactly one effect, the bug is not observable. Users requesting unbounded exports (`limit < 0`) also do not see this discrepancy because no row cap is expected.
