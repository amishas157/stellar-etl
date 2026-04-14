# H002: `export_ledgers --captive-core` drops evicted-ledger-key arrays

**Date**: 2026-04-14
**Subsystem**: cli-commands
**Severity**: High
**Impact**: missing eviction metadata in ledger exports
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a ledger's `LedgerCloseMeta` contains `EvictedKeys`, then `export_ledgers` should emit matching `evicted_ledger_keys_type` and `evicted_ledger_keys_hash` arrays regardless of whether the operator requested the default datastore reader or `--captive-core`.

## Mechanism

The `--captive-core` branch in `export_ledgers` bypasses the ledger-backend path and reads plain history-archive ledgers, which carry no `LedgerCloseMeta` payload in `HistoryArchiveLedgerAndLCM.LCM`. `TransformLedger()` only derives `evicted_ledger_keys_type/hash` from `lcm.GetV1()` / `lcm.GetV2()`, so ledgers that really evicted keys are exported with empty arrays on the `--captive-core` path.

## Trigger

Run `export_ledgers --captive-core` over a ledger range containing at least one ledger with non-empty `EvictedKeys` in close meta. The exported row for that ledger will show empty/null `evicted_ledger_keys_type` and `evicted_ledger_keys_hash`, while the same ledger exported through the normal backend path can populate those arrays.

## Target Code

- `cmd/export_ledgers.go:28-32` — routes `--captive-core` through the history-archive helper
- `internal/input/ledgers_history_archive.go:GetLedgersHistoryArchive:10-34` — leaves `LCM` unset for every returned ledger
- `internal/transform/ledger.go:63-90` — only populates eviction metadata from `LedgerCloseMeta` V1/V2
- `internal/transform/ledger.go:125-128` — writes the empty `evicted_ledger_keys_*` slices into the final `LedgerOutput`

## Evidence

`TransformLedger()` initializes `outputEvictedKeysHash` and `outputEvictedKeysType` as nil slices and only calls `transformLedgerKeys()` inside the `lcm.GetV1()` / `lcm.GetV2()` branches. Because the history-archive path never populates `LCM`, those branches never run under `--captive-core`, even though real close meta can carry `EvictedKeys`.

## Anti-Evidence

Many ledgers do not evict any keys, so empty arrays are sometimes correct. The bug only appears on ledgers where eviction metadata is present in close meta, but on those ledgers the wrong output is still success-shaped and plausible.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`
**Date**: 2026-04-14
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`
**Failed At**: reviewer

### Trace Summary

The hypothesis describes the evicted-keys aspect of an already-confirmed, broader finding. `success/data-input/001` documents the exact same root cause: `GetLedgersHistoryArchive()` never sets `LCM`, so `TransformLedger()` never enters the `lcm.GetV1()`/`lcm.GetV2()` branches that populate Soroban fields. That confirmed finding explicitly covers evicted keys — its PoC test asserts on `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` (lines 78-90 of the PoC). The fail file `035-captive-core-ledgers-zero-soroban-metrics.md` was also rejected as a duplicate of the same success entry.

### Code Paths Examined

- `cmd/export_ledgers.go:28-32` — confirmed `UseCaptiveCore` routes to `GetLedgersHistoryArchive()`
- `internal/input/ledgers_history_archive.go:24-26` — confirmed `LCM` field is never set
- `internal/transform/ledger.go:61-91` — confirmed evicted keys only populated from `lcm.GetV1()`/`lcm.GetV2()` branches (lines 73, 87)

### Why It Failed

This is an exact duplicate of `success/data-input/001-export-ledgers-captive-core-drops-soroban-lcm-fields.md.gh-published`, which already covers ALL LCM-derived fields being dropped on the `--captive-core` path — including `EvictedLedgerKeysType` and `EvictedLedgerKeysHash` (explicitly tested in its PoC). The present hypothesis merely narrows focus to the evicted-keys subset of the same bug.

### Lesson Learned

The `--captive-core` zero-LCM bug in `export_ledgers` drops ALL close-meta-derived fields (Soroban metrics, evicted keys, etc.) as a single root cause. Hypotheses targeting individual fields within this set are subsumed by the existing finding. Check `success/data-input/001` before proposing any captive-core LCM field hypothesis.
