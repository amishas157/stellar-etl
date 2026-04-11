# H002: Transaction-scoped contract events serialize NULL operation IDs as `0` in parquet

**Date**: 2026-04-11
**Subsystem**: toid
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For transaction-level contract events and diagnostic events that are not attached to a specific operation, parquet output should preserve `operation_id = NULL`, matching the JSON export and indicating that no operation TOID exists for that row.

## Mechanism

`TransformContractEvent()` appends transaction-level and diagnostic events without setting `OperationID`, so the source value is intentionally null. However, the parquet side models `operation_id` as a required `int64` instead of a nullable field, and the converter provides no validity check. As a result, rows with no operation context are serialized as `0`, manufacturing a fake TOID instead of preserving the absence of one.

## Trigger

Run `export_contract_events --write-parquet` on a ledger containing a transaction-scoped contract event or diagnostic event emitted outside `transactionEvents.OperationEvents`.

## Target Code

- `internal/transform/contract_events.go:30-39` — transaction-level events are appended without assigning `OperationID`.
- `internal/transform/contract_events.go:58-65` — diagnostic events are appended without assigning `OperationID`.
- `internal/transform/schema.go:641-657` — the source schema declares `OperationID null.Int`.
- `internal/transform/schema_parquet.go:382-399` — the parquet schema narrows that nullable field to plain `int64`.
- `internal/transform/parquet_converter.go:425-441` — parquet conversion does not preserve null validity.

## Evidence

The transform code uses `null.Int` specifically because some contract-event rows have no operation TOID. On the parquet side there is no nullable wrapper, no optional tag, and no branch that distinguishes `Valid=false` from `Int64==0`, so the semantic difference between "no operation" and "operation ID zero" is lost.

## Anti-Evidence

There is no real Stellar operation whose canonical TOID is `0`, so consumers may be able to recognize the bad rows after the fact. But the export itself is still wrong because it turns a null relationship into a numeric value.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of success/toid/001-contract-event-parquet-drops-operation-toid; also describes working-as-designed behavior
**Failed At**: reviewer

### Trace Summary

Traced the `ContractEventOutput.ToParquet()` method at `parquet_converter.go:425-441`, which currently omits `OperationID` entirely (the root cause of success finding 001). Then audited all parquet converters in the same file and confirmed that every `null.Int` field is systematically converted via `.Int64` (yielding `0` for null) and every `null.String` via `.String` (yielding `""` for null). The parquet schema never uses `OPTIONAL` repetition type for scalar fields — this is a deliberate, codebase-wide design pattern, not a bug specific to `operation_id`.

### Code Paths Examined

- `internal/transform/parquet_converter.go:425-441` — `ContractEventOutput.ToParquet()` omits `OperationID` entirely (already captured by success 001)
- `internal/transform/parquet_converter.go:84-86` — `TransactionOutput.ToParquet()` uses `.Int64` for `MinAccountSequence`, `MinAccountSequenceAge`, `MinAccountSequenceLedgerGap` (all `null.Int` → bare `int64`)
- `internal/transform/parquet_converter.go:122` — `AccountOutput.ToParquet()` uses `.String` for `Sponsor` (`null.String` → bare `string`)
- `internal/transform/parquet_converter.go:266-272` — `TradeOutput.ToParquet()` uses `.Int64` for `SellingOfferID`, `BuyingOfferID`, `LiquidityPoolFee`, `RoundingSlippage` (all `null.Int` → bare `int64`)
- `internal/transform/schema_parquet.go` — zero uses of `OPTIONAL` repetition type for scalar fields; only `REPEATED` for arrays

### Why It Failed

Two independent reasons:

1. **Duplicate of success finding 001**: The already-confirmed finding `success/toid/001-contract-event-parquet-drops-operation-toid` documents the same `ToParquet()` omission. The fix for 001 (`OperationID: ceo.OperationID.Int64`) would follow the established `.Int64` pattern, which is exactly the null→0 behavior this hypothesis calls a bug.

2. **Working as designed**: The hypothesis treats the null→0 parquet conversion as a bug, but this is a deliberate, systematic design choice. Every `null.Int` in the JSON schema becomes a bare `int64` in the parquet schema across all 10+ converters. No parquet field uses `OPTIONAL` repetition type for nullable scalars. The hypothesis misidentifies a consistent architectural decision as a defect specific to `operation_id`.

### Lesson Learned

Before claiming a parquet field should be nullable, check whether *any* parquet field in the codebase uses nullable semantics. If zero fields do, the null→zero-value mapping is an intentional design trade-off, not a per-field bug. Also check success findings before investigating — the underlying code path was already captured.
