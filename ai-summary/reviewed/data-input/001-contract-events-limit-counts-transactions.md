# H001: Contract-event export applies `--limit` to transactions instead of emitted events

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty or truncated contract-event exports under caller-visible row limits
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_contract_events --limit N` should keep scanning the requested ledger range until it has emitted `N` `ContractEventOutput` rows or exhausted the range. Transactions that produce zero contract-event rows should not consume the caller's event budget.

## Mechanism

`export_contract_events` passes the caller-visible `limit` directly into `input.GetTransactions()`, and that reader stops after `N` transactions rather than after `N` exported contract events. `transform.TransformContractEvent()` then expands each returned transaction into zero, one, or many output rows by concatenating transaction events, operation events, and diagnostic events, so the command can silently return an empty or truncated file even when later transactions in-range contain real contract events.

## Trigger

Run `stellar-etl export_contract_events --limit 1 --start-ledger <S> --end-ledger <E>` on a range where the first transaction is a classic/non-event transaction but a later transaction in the same range emits Soroban contract or diagnostic events. The correct output is one event row from the later transaction; the current code can return zero rows because the first transaction consumed the entire limit before expansion.

## Target Code

- `cmd/export_contract_events.go:25-43` — applies the user-visible `limit` at `GetTransactions()` time and then exports every event row returned by the transform
- `internal/input/transactions.go:23-70` — enforces `limit` on `len(txSlice)`, i.e. transaction count
- `internal/transform/contract_events.go:21-67` — expands one transaction into a variable-length `[]ContractEventOutput`
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `contract_events` to export
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction.go:278-311` — `GetTransactionEvents()` legitimately returns empty event sets for many transactions

## Evidence

The flag text is row-oriented (`contract_events`), but the reader budget is transaction-oriented. `TransformContractEvent()` explicitly appends rows from three independent event collections, while the upstream SDK's `GetTransactionEvents()` returns empty results for many transactions (for example, TxMeta V1/V2 and non-Soroban V3 paths), so an early zero-event transaction can exhaust `--limit` before any event row is emitted.

## Anti-Evidence

If the first limited transactions all happen to emit at least one contract-event row, the command still produces data and the bug is easy to miss. Users who leave `--limit` negative also avoid the problem because the reader scans the full range.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The hypothesis accurately describes a real limit-granularity mismatch in `export_contract_events`. The command passes the user-visible `--limit` (documented as "Maximum number of contract_events to export") directly into `input.GetTransactions()`, which enforces it at transaction count granularity. `TransformContractEvent()` then expands each transaction into zero-to-many `ContractEventOutput` rows from three separate event arrays, but the exporter writes all returned rows without any secondary cap. This is the same class of bug confirmed in `export_effects` (success/data-input/004) and `export_trades` (success/export-pipeline/010), but applied to a distinct command with its own independent code path.

### Code Paths Examined

- `cmd/export_contract_events.go:25` — passes `cmdArgs.Limit` directly to `input.GetTransactions()`, applying the limit at transaction granularity
- `cmd/export_contract_events.go:33-53` — iterates ALL events from `TransformContractEvent()` without any secondary row count cap
- `internal/input/transactions.go:51` — `for int64(len(txSlice)) < limit || limit < 0` enforces limit on `len(txSlice)` (transaction count)
- `internal/input/transactions.go:65-67` — breaks outer ledger loop when transaction limit is reached
- `internal/transform/contract_events.go:21-67` — `TransformContractEvent` calls `transaction.GetTransactionEvents()` and concatenates rows from `TransactionEvents`, `OperationEvents`, and `DiagnosticEvents` arrays — any of which may be empty for non-Soroban transactions
- `internal/utils/main.go:254` — flag definition: `"Maximum number of contract_events to export"` — clearly specifies event-row semantics

### Findings

The bug manifests in two directions:

1. **Empty/truncated exports**: With `--limit N` on a mixed ledger range, if the first N transactions are classic (non-Soroban), `GetTransactions()` returns N transactions that each produce zero contract events. The exporter writes zero rows even though later in-range Soroban transactions have events. This is especially likely because most Stellar transactions are classic, not Soroban.

2. **Oversized exports**: If one of the first N transactions is a complex Soroban invocation emitting many events, the exporter writes all of them with no cap, exceeding the user's requested limit.

The contract events case is arguably more impactful than the effects case (success/data-input/004) because a much higher fraction of transactions produce zero events (only Soroban transactions emit contract events), making the empty-export scenario far more likely in practice.

Severity downgraded from High to Medium for consistency with the confirmed effects finding (data-input/004, also Medium), since both are the same class of operational correctness issue with the `--limit` flag. Users running production pipelines typically use `--limit -1` (the default), which bypasses this codepath entirely.

### PoC Guidance

- **Test file**: `internal/transform/contract_events_test.go` (or create a new test in `internal/transform/data_integrity_poc_test.go` if it exists)
- **Setup**: Build a `LedgerTransaction` with `TxMeta` V1 or V2 (no Soroban events), similar to the existing test fixtures in `contract_events_test.go`. Confirm `TransformContractEvent()` returns an empty slice.
- **Steps**: (1) Call `TransformContractEvent()` on a non-Soroban transaction and verify it returns 0 rows. (2) Show that `GetTransactions(start, end, limit=1, ...)` returns exactly 1 transaction regardless of event content. (3) Demonstrate that the exporter loop writes all returned events without a secondary cap.
- **Assertion**: Assert that a single non-Soroban transaction passed through `TransformContractEvent()` yields `len(result) == 0`, proving that `--limit 1` would produce an empty export when the first transaction has no events, even though the flag promises "Maximum number of contract_events to export."
