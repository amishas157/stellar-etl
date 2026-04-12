# H001: `export_assets --limit` overruns the requested asset count within the first ledger

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural row-count corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an operator runs `export_assets --limit N`, the command should emit at most `N` asset rows, because the shared archive flag is documented as the maximum number of exported `assets`. A range capped at `--limit 5` should therefore stop after writing five asset records, not a larger success-shaped file.

## Mechanism

`export_assets` forwards `limit` into `input.GetPaymentOperations()`, but that helper only checks `len(assetSlice) >= limit` after it finishes scanning an entire ledger. If the first ledger in range already contains more than `N` qualifying payment/manage-sell operations, the helper returns all of them, and `export_assets` writes each transformed asset row before reporting success.

Because the limit enforcement happens at the ledger boundary instead of the exported-row boundary, the command can silently over-emit assets. The existing `-l 5` regression fixture already demonstrates this: the accepted golden output contains 21 asset rows for the supposedly five-row export.

## Trigger

Run:

`stellar-etl export_assets -s 30822015 -e 30822025 -l 5 -o /tmp/assets.json`

The current checked-in fixture for this command path (`cmd/export_assets_test.go`'s `"range too large"` case) accepts 21 output rows, because ledger `30822015` alone contributes more than five qualifying asset-bearing operations before the post-ledger limit check runs.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — public flag contract says `limit` is the maximum number of exported `assets`
- `cmd/export_assets.go:21-45` — passes the user limit into `GetPaymentOperations()`
- `cmd/export_assets.go:53-70` — writes every returned asset row with no additional limit enforcement
- `internal/input/assets.go:GetPaymentOperations:20-58` — appends every qualifying operation in a ledger, then checks `limit` only after the ledger is fully scanned
- `cmd/export_assets_test.go:22-25` — existing `"range too large"` CLI case exercises `-l 5`

## Evidence

`GetPaymentOperations()` appends qualifying operations inside the nested transaction/operation loops, but its only `limit` guard sits at lines 55-56 after the whole ledger finishes. The checked-in `testdata/assets/large_range_assets.golden` for the `-l 5` case contains 21 JSON rows, and every visible row in the fixture comes from the very first ledger (`30822015`), matching the "finish ledger first, enforce limit later" mechanism.

## Anti-Evidence

The command's internal comment block describes `limit` as the maximum number of operations to export, which suggests some developers may have been thinking in terms of input operations rather than final asset rows. But the user-facing flag text comes from `AddArchiveFlags("assets", ...)`, so the CLI contract exposed to operators is still "maximum number of assets to export," making the current output shape a correctness bug rather than a mere implementation detail.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the limit enforcement path from `AddArchiveFlags` through `GetPaymentOperations` (assets.go:55) and `GetPaymentOperationsHistoryArchive` (assets_history_archive.go:45). Both functions check the limit only at ledger boundaries, after collecting all qualifying operations from the current ledger. Compared this pattern with sibling functions: `operations.go` (lines 52, 67) and `trades.go` (line 73) both enforce limits inside inner loops, while assets.go does not. The golden test `large_range_assets.golden` explicitly accepts 21 rows for a `-l 5` invocation.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:250-254` — flag text says "Maximum number of assets to export"
- `internal/input/assets.go:GetPaymentOperations:20-61` — limit checked at line 55 AFTER inner loops complete; no inner-loop guard
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:13-51` — identical ledger-boundary-only limit pattern
- `internal/input/operations.go:52-77` — has BOTH pre-loop guard (line 52) and inner-loop break (line 67)
- `internal/input/trades.go:73-81` — has inner-loop break (line 73) plus post-ledger break (line 80)
- `cmd/export_assets.go:44-70` — no secondary limit check on output rows
- `testdata/assets/large_range_assets.golden` — 21 rows accepted as golden output for `-l 5`

### Why It Failed

The hypothesis correctly identifies that `assets.go` lacks inner-loop limit checks present in sibling functions (`operations.go`, `trades.go`). However, this does not constitute data correctness corruption under the investigation objective:

1. **All output row values are correct.** Every asset row contains the right asset_code, asset_id, asset_issuer, asset_type, closed_at, and ledger_sequence. No field holds a wrong value.
2. **The golden test defines accepted behavior.** The test suite's `large_range_assets.golden` explicitly accepts 21 rows for `-l 5`, making this known and approved behavior — not a deviation from the code's expected behavior. The golden file IS the source of truth for correctness.
3. **Extra correct data is not corruption.** Downstream systems (BigQuery, analytics) process variable row counts without issue. Receiving 21 correct rows instead of 5 does not cause wrong calculations, hash collisions, or compliance violations.
4. **The limit is a soft operational threshold, not a data integrity constraint.** Ledger-boundary enforcement is a reasonable design trade-off for a ledger-oriented tool. The flag text is imprecise documentation, not a tested data invariant.
5. **Consistent with established rejection pattern.** Fail entries 016 and 018 in this subsystem rejected similar hypotheses that treated export_assets documentation imprecision as data bugs.

### Lesson Learned

Pattern 5 (loop condition consistency) correctly identifies that `assets.go` is missing inner-loop limit checks present in `operations.go` and `trades.go`. However, when the test suite's golden output explicitly validates the "overrun" behavior, the discrepancy is a documentation imprecision, not a data correctness bug. For this objective, the question is whether OUTPUT VALUES are wrong — not whether an operational flag is enforced at a coarser granularity than its help text implies.
