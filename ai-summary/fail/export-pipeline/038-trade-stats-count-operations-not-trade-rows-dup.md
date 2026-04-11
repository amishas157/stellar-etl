# H003: `export_trades` completion stats count trade-capable operations instead of exported trade rows

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

The final `export_trades` stats should describe the number of trade rows actually emitted, so operators can compare the completion log against the size of the produced trade dataset.

## Mechanism

`TransformTrade()` can emit multiple `TradeOutput` rows for a single operation when that operation claims multiple offers, but `export_trades` reports `PrintTransformStats(len(trades), numFailures)` where `trades` is the pre-expansion slice of trade-capable operations. A single path payment or offer op that crosses several offers can therefore write multiple trade rows while only incrementing the reported success count once.

This creates a quiet but persistent mismatch between the post-write stats JSON and the exported trade file. Monitoring that treats `successful_transforms` as an emitted-row count will systematically undercount ranges with multi-claim operations.

## Trigger

Run `export_trades` on a ledger range containing a path payment or offer operation that claims multiple offers. The output file will contain one row per claim atom, but the final stats will still count only the single source operation.

## Target Code

- `cmd/export_trades.go:34-58` — expands each `TradeTransformInput` into `[]TradeOutput` and writes every row
- `cmd/export_trades.go:64-70` — reports stats from `len(trades)` rather than emitted trade rows
- `internal/transform/trade.go:41-160` — `TransformTrade()` appends one `TradeOutput` per claimed offer
- `cmd/command_utils.go:89-102` — helper assumes attempts and successes are measured in the same unit

## Evidence

`TransformTrade()` iterates `claimedOffers` and appends a distinct `TradeOutput` per claim. The command later logs only the number of `TradeTransformInput` items, not the total number of `TradeOutput` rows written.

## Anti-Evidence

As with the effects path, reviewers may decide the stats intentionally measure source-operation transforms rather than output rows. The issue is only meaningful if the final stats log is supposed to represent exported trade-row counts.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — exact duplicate of fail/export-pipeline/036-trade-stats-count-operations-not-trade-rows.md
**Failed At**: reviewer

### Trace Summary

This hypothesis is an exact duplicate of an already-reviewed and rejected investigation at `ai-summary/fail/export-pipeline/036-trade-stats-count-operations-not-trade-rows.md`. The prior review comprehensively traced all 9 export commands and confirmed that `PrintTransformStats` consistently counts at input-item granularity across the entire codebase — this is working-as-designed behavior, not a bug.

### Code Paths Examined

- `cmd/export_trades.go:28-64` — already examined in fail/036
- `cmd/command_utils.go:90-103` — already examined in fail/036

### Why It Failed

**Duplicate of fail/export-pipeline/036-trade-stats-count-operations-not-trade-rows.md.** The prior review established that `PrintTransformStats` is an input-counting convention across all export commands, including all four 1:N fan-out exporters (trades, effects, contract_events, token_transfers). This is a codebase-wide architectural decision, not a per-command bug. See also fail/035 (effects variant) and fail/034 (assets variant) for the same class of hypothesis.

### Lesson Learned

This exact hypothesis was already investigated. The stats-counting convention is documented across multiple fail files (034, 035, 036) and in the summary.md meta-patterns.
