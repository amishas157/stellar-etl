# H001: `export_assets` success stats count duplicate operations instead of emitted asset rows

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_assets` finishes, the final `attempted_transforms` / `successful_transforms` log should match the number of asset rows the command actually emitted, so operators can use that summary as a completion signal for the exported dataset.

## Mechanism

`export_assets` transforms one payment/manage-sell operation at a time, but then deduplicates rows by `asset_id` before writing. The command still calls `PrintTransformStats(len(paymentOps), numFailures)`, so duplicate-heavy ranges can report many successful transforms even when only a small number of distinct asset rows were actually written.

This becomes meaningful because `export_assets` already uses the log summary after the write loop as its public completion signal. A range dominated by repeated native-asset payments can therefore look healthy in logs while silently emitting far fewer rows than the stats imply.

## Trigger

Run `export_assets` on any ledger range with many repeated references to the same asset, such as multiple native-asset payment operations in the same range. The JSON output will contain one row per distinct asset, but the final `successful_transforms` count will still reflect the full pre-dedup `paymentOps` slice.

## Target Code

- `cmd/export_assets.go:39-75` — deduplicates by `seenIDs` but reports stats from `len(paymentOps)`
- `cmd/command_utils.go:89-102` — computes `successful_transforms = attempts - failures` without visibility into post-transform dedupe

## Evidence

The exporter explicitly skips already-seen assets at `export_assets.go:53-56`, after a successful transform but before `ExportEntry()`. Nothing adjusts the final stats to the number of rows that actually reached the file.

## Anti-Evidence

The stats helper is named `PrintTransformStats`, so reviewers may decide the contract is intentionally source-operation-oriented rather than output-row-oriented. The bug is only real if those final counters are meant to describe the exported dataset rather than the pre-dedup transform loop.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (related success/018 covers the `--limit` underfill, not stats reporting)
**Failed At**: reviewer

### Trace Summary

Traced `PrintTransformStats` usage across all 10 export commands. Every command passes `len(inputSlice)` as the attempts parameter — `export_ledgers`, `export_transactions`, `export_operations`, `export_effects`, `export_trades`, `export_assets`, `export_contract_events`, `export_token_transfers`, `export_ledger_transaction`, and `export_ledger_entry_changes`. None adjust for post-transform filtering or fan-out. The function's name (`PrintTransformStats`) and its uniform usage establish a clear contract: it reports transform-layer throughput, not output-row counts.

### Code Paths Examined

- `cmd/command_utils.go:PrintTransformStats:90-103` — computes `successful_transforms = attempts - failures`; documented as "the number of attempted, failed, and successful transformations"
- `cmd/export_assets.go:44-70` — transform loop with dedup via `seenIDs`; skipped duplicates are not counted as failures
- `cmd/export_assets.go:75` — `PrintTransformStats(len(paymentOps), numFailures)` passes raw input count
- `cmd/export_operations.go:59` — `PrintTransformStats(len(operations), numFailures)` — same pattern, no dedup
- `cmd/export_trades.go:64` — `PrintTransformStats(len(trades), numFailures)` — same pattern
- `cmd/export_assets.go:73` — `cmdLogger.Infof("%d bytes written to %s", totalNumBytes, outFile.Name())` — separate output-level signal

### Why It Failed

**Describes working-as-designed behavior.** `PrintTransformStats` has a consistent, codebase-wide contract: it counts input items as transform attempts and subtracts failures. All 10 export commands follow this exact pattern. The function name says "Transform Stats", not "Export Stats" or "Output Stats." The dedup step in `export_assets` occurs after transformation — the transforms genuinely succeed for duplicates; they are just not exported. A separate `bytes written` log line (line 73) provides an output-level completion signal. The hypothesis misinterprets the stats function's contract as output-row-oriented when it is transform-operation-oriented by design.

### Lesson Learned

`PrintTransformStats` is an input-counting convention across all commands. Hypotheses about its stats diverging from output row counts will always fail because the function intentionally counts transform operations, not exported rows. The separate `bytes written` log provides the output-level signal.
