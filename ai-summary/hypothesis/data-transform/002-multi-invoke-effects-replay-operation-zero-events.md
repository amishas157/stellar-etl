# H002: Multi-invoke effects replay operation-zero contract events

**Date**: 2026-04-11
**Subsystem**: data-transform
**Severity**: Critical
**Impact**: financial field produces wrong numeric value
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For a Soroban transaction containing multiple `InvokeHostFunction` operations, each exported `history_effects` row should be derived from that specific operation's contract events and should carry the correct `operation_id`, amount, asset, and source/recipient details for that operation only. Later invoke operations must not reuse earlier operations' SAC events.

## Mechanism

`TransformEffect()` handles every `InvokeHostFunction` operation by calling `operation.transaction.GetContractEvents()`. The SDK helper is explicitly documented as a Soroban convenience that assumes there is only one operation in the transaction and always returns `GetContractEventsForOperation(0)`. As a result, once a transaction contains two invoke operations, the exporter feeds operation 0's contract events into every later invoke operation, so later rows are tagged with the later `operation_id` but contain operation 0's amounts/contracts, while the real later-operation effects are omitted entirely.

## Trigger

Construct or export a Soroban transaction with two `InvokeHostFunction` operations that emit distinct SAC transfer-style events, then run `export_effects`. The rows for operation 2 will carry operation 2's `operation_id`, but their `details.amount`, asset fields, and contract/account endpoints will match operation 1's events; operation 2's real effects will be missing.

## Target Code

- `internal/transform/effects.go:118-130` — invoke-host-function effects source events from `transaction.GetContractEvents()`
- `internal/transform/effects.go:1322-1433` — `addInvokeHostFunctionEffects()` turns the supplied event set into `history_effects` rows for the current operation
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go@v0.0.0-20250915171319-4914d3d0af61/ingest/ledger_transaction.go:245-257` — `GetContractEvents()` is hard-wired to operation index 0 and documents the one-operation assumption

## Evidence

The event source is transaction-scoped and operation-agnostic, but the emitted effect rows use the current wrapper's `OperationID`, so a wrong event slice produces plausible but misattributed rows. The existing invoke-host-function effect tests only exercise a single-operation V3 transaction, which matches the helper's stale assumption and leaves the multi-operation case uncovered. External Stellar discussion confirms core now supports multiple `InvokeHostFunction` operations in one Soroban transaction as long as they are not mixed with classic ops.

## Anti-Evidence

Older Soroban tooling and the SDK helper comment still assume one invoke operation per transaction, which can hide this corruption if exported ledgers never include multi-invoke Soroban transactions. Single-operation invoke transactions continue to export correctly.
