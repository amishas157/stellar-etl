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
