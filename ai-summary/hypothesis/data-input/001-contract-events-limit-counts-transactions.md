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
