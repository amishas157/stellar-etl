# H007: Token-transfer parquet TOID loss looked possible, but no parquet path exists today

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If token transfers are exported to parquet, their `operation_id` should preserve the same nullable TOID semantics as JSON: operation-scoped rows should contain the operation TOID, and transaction-scoped rows should remain null.

## Mechanism

At first glance, token transfers looked like another nullable-TOID parquet mismatch because `TokenTransferOutput.OperationID` is a `null.Int` and the transform package has no `TokenTransferOutputParquet` or `ToParquet()` implementation. That suggested the parquet path might be silently defaulting missing TOIDs to `0` or dropping the field entirely.

## Trigger

Run `export_token_transfer --write-parquet` on a ledger containing token-transfer events with and without `OperationIndex`.

## Target Code

- `internal/transform/schema.go:659-677` — token-transfer JSON schema uses `OperationID null.Int`.
- `cmd/export_token_transfers.go:19-23` — command parses common flags but discards the archive `parquet-output` return value.
- `cmd/export_token_transfers.go:38-63` — command only writes JSON rows and never calls `WriteParquet(...)`.
- `internal/utils/main.go:231-255` — archive commands still inherit `--write-parquet` and `--parquet-output`.
- `internal/utils/main.go:458-537` — `MustCommonFlags` does parse `WriteParquet`.

## Evidence

The command advertises the shared archive/common flag surface, and the transform package lacks any token-transfer parquet schema/converter. That combination initially looked like a live TOID-corruption risk on the parquet path.

## Anti-Evidence

A repo-wide search found no `TokenTransferOutputParquet` type and no `TokenTransferOutput.ToParquet()` implementation, and `export_token_transfers.go` never checks `commonArgs.WriteParquet` or calls `WriteParquet(...)`. The current code only emits JSON for token transfers.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

There is no current parquet export path for token transfers, so there is no live parquet file in which TOIDs can be silently corrupted. The bug candidate is a missing feature / ignored flag, not an existing data-integrity defect in exported parquet rows.

### Lesson Learned

Before classifying a nullable-ID mismatch as live corruption, confirm that the command actually writes that dataset through `WriteParquet(...)`. Shared `--write-parquet` flags are not enough on their own if the command never exercises the parquet converter path.
