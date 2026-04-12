# H014: Batch compaction should collapse duplicate keys across the whole ledger-change batch

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: Medium
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `extractBatch()` is supposed to emit the final state for the inclusive batch `[batchStart, batchEnd]`, repeated updates to the same ledger key inside that batch should collapse to one caller-visible row instead of multiple intermediate states.

## Mechanism

`extractBatch()` describes itself as compacting “the changes from the ledgers in the range [batchStart, batchEnd]”, but it re-creates every `ChangeCompactor` inside the per-ledger loop. That initially suggests the batch may keep stale intermediate account / trustline / offer states rather than one final state per key.

## Trigger

1. Inspect a multi-ledger change-export batch where the same ledger entry is updated in consecutive ledgers inside one exported batch window.
2. Check whether the batch contains multiple rows for that key instead of one final compacted row.

## Target Code

- `internal/input/changes.go:82-157` — `extractBatch()` rebuilds `changeCompactors` inside the `for seq := batchStart; seq <= batchEnd` loop
- `cmd/export_ledger_entry_changes.go:111-260` — exports every change collected in the batch without additional deduplication

## Evidence

The implementation comment and function name use “batch” language, and the per-ledger compactor reset looks like a plausible source of duplicate intermediate rows across a wider batch.

## Anti-Evidence

Upstream `ingest.ChangeCompactor` is explicitly documented as compacting “all changes within a single ledger” and warns that it is designed for one-ledger `LedgerCloseMeta` streams, not cross-ledger state folding. That makes the per-ledger reset a deliberate contract match rather than a bug.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The export path is intentionally preserving per-ledger change history inside each batch window, and the upstream compactor API is only defined for single-ledger normalization. Reusing one compactor across multiple ledgers would violate that contract rather than fix a real exporter bug.

### Lesson Learned

When a helper claims to “compact” a batch, verify the upstream abstraction boundary before assuming full-window deduplication. In this pipeline, “batch” groups ledgers for I/O efficiency, while compaction still happens one ledger at a time by design.
