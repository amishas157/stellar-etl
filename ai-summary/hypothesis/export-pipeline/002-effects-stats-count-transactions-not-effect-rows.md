# H002: `export_effects` completion stats count source transactions instead of exported effect rows

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

After `export_effects` writes its output, the final stats log should reflect the number of effect rows actually exported, because downstream batch monitoring uses that summary to judge completeness of the effect dataset.

## Mechanism

`TransformEffect()` expands a single successful transaction into zero, one, or many `EffectOutput` rows, but `export_effects` still reports `PrintTransformStats(len(transactions), numFailures)`. A transaction that generates several effects therefore contributes only one reported success even though multiple rows were written, while a quiet range can report the same success count as a high-fanout range with many more emitted effects.

That mismatch is operationally significant because it makes the final stats JSON unusable as a row-count signal for the exported file. The command logs the stats after the write loop, which makes them look like output counters even though they are really transaction counters.

## Trigger

Run `export_effects` on a ledger range containing a successful transaction that emits multiple classic effects, such as `create_account`. The output file will contain multiple effect rows derived from that one transaction, but the final `successful_transforms` value will increase by only one.

## Target Code

- `cmd/export_effects.go:31-56` — expands each transaction into `effects []EffectOutput` and writes every row
- `cmd/export_effects.go:62-68` — reports stats from `len(transactions)` after row-level export is complete
- `cmd/command_utils.go:89-102` — generic stats helper assumes a 1:1 attempt-to-success model

## Evidence

The exporter loops over `transactions`, then nests a second loop over the `effects` slice returned by `TransformEffect()`. The final stats call ignores the size of that emitted slice and only reports the number of source transactions.

## Anti-Evidence

This may be treated as intentional if the project considers “transforms” to mean source-transaction transforms rather than emitted effect rows. The hypothesis depends on the post-write stats being intended as dataset-level counters for the exported effects file.
