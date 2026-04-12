# H030: `TransformPool()` looked like it could emit an empty liquidity-pool row, but the current caller makes that branch unreachable

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `export_ledger_entry_changes` writes a `liquidity_pools` row at all, it should come from a real liquidity-pool ledger entry rather than an all-zero `PoolOutput{}` shell.

## Mechanism

`TransformPool()` contains a skip branch that returns `PoolOutput{}, nil` when the extracted ledger entry is not a liquidity pool. That looked like a silent empty-row risk, but the only production caller first switches on `entryType == xdr.LedgerEntryTypeLiquidityPool` before invoking `TransformPool()`, so non-pool entries never reach the helper in the live export path.

## Trigger

1. Run `export_ledger_entry_changes` over a mixed batch containing many ledger-entry types.
2. Follow the `liquidity_pools` export branch.
3. Confirm that only `LedgerEntryTypeLiquidityPool` changes call `TransformPool()`.

## Target Code

- `internal/transform/liquidity_pool.go:TransformPool:13-22` — helper returns `PoolOutput{}, nil` for non-pool entries
- `cmd/export_ledger_entry_changes.go:199-210` — caller only invokes `TransformPool()` inside the liquidity-pool entry-type case

## Evidence

The helper itself has the silent skip branch, so the concern is real at the function level. But `export_ledger_entry_changes` iterates `batch.Changes` by `entryType` and only enters the pool loop for `xdr.LedgerEntryTypeLiquidityPool`, which means the helper's non-pool guard is dead in the current production path.

## Anti-Evidence

A future caller could misuse `TransformPool()` directly and hit the empty return, so the helper is not universally safe. The user-facing export path being investigated here, however, already prevents that misuse.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The apparent empty-row bug depends on non-liquidity-pool entries reaching `TransformPool()`, but the current export command filters by `entryType` first, so that branch is unreachable in production.

### Lesson Learned

Helper-level silent skips are not automatically live bugs. Always trace the production caller to see whether upstream dispatch already makes the suspicious branch impossible.
