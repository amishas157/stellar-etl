# H054: `export_token_transfer --strict-export` still logs and continues on row errors

**Date**: 2026-04-12
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: silent partial export
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `--strict-export` is enabled, any transform or write failure in `export_token_transfer` should abort the command immediately rather than logging the error and continuing with a partial output file. A strict run should either complete fully or fail hard.

## Mechanism

`export_token_transfer` sets `cmdLogger.StrictExport` but still handles transform and export failures through `cmdLogger.LogError(...)` inside the main loop. That initially looked like a silent-partial-export bug because the command body itself does not `return` or `Fatal` after those errors.

## Trigger

1. Run `export_token_transfer --strict-export` on an input that makes `TransformTokenTransfer()` or `ExportEntry()` fail.
2. Observe whether the command continues iterating and produces a partial file instead of aborting.

## Target Code

- `cmd/export_token_transfers.go:18-24` — copies `--strict-export` into `cmdLogger.StrictExport`
- `cmd/export_token_transfers.go:38-52` — routes transform/export failures through `cmdLogger.LogError`
- `internal/utils/logger.go:17-23` — `LogError` decides whether strict mode logs or fatals

## Evidence

The command loop does not contain any local `return`, `break`, or `Fatal` in its error branches; it only increments `numFailures` after calling `cmdLogger.LogError(...)`. That makes the path look like a likely strict-mode omission when read in isolation.

## Anti-Evidence

`EtlLogger.LogError()` is the shared strict-export gate, and it calls `l.Fatal(err)` whenever `StrictExport` is true. So the command already aborts on the first logged error in strict mode even though the loop body itself uses the generic helper.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

Strict mode is already enforced centrally by `EtlLogger.LogError()`. Once `export_token_transfer` copies `commonArgs.StrictExport` into the logger, any subsequent `LogError(...)` path fatals instead of merely logging, so the command does not silently continue under `--strict-export`.

### Lesson Learned

Do not judge strict-export behavior from a single command file alone when the repo funnels error handling through a shared logger wrapper. If a command uses `cmdLogger.LogError`, inspect `internal/utils/logger.go` before assuming strict mode is inert.
