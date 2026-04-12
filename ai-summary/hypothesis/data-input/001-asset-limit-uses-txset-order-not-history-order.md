# H001: `export_assets --limit` selects asset rows in tx-set order instead of canonical history order

**Date**: 2026-04-12
**Subsystem**: data-input
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_assets --limit N` is used, the command should emit the first `N` qualifying asset-discovery rows in the same transaction application order used by the rest of the history exports. A limited export over a ledger with multiple qualifying operations should therefore be a prefix of the canonical history order, not a prefix of the raw transaction-set envelope order.

## Mechanism

Both asset readers bypass `LedgerTransactionReader` and iterate raw envelopes from `LedgerCloseMeta.TransactionEnvelopes()` / `transform.GetTransactionSet(ledger)` directly. The SDK reader comment explains that envelope order in the agreed-upon transaction set can differ from the processed meta order because transaction metas are sorted by hash; since `GetPaymentOperations*()` applies the `limit` while scanning this raw order, `export_assets --limit` can silently choose a different subset of rows than the canonical history prefix that other exports use.

## Trigger

1. Find a ledger where two qualifying `payment` or `manage_sell_offer` operations appear in different raw tx-set order vs processed transaction order.
2. Run `stellar-etl export_assets --start-ledger <L> --end-ledger <L> --limit 1`.
3. Compare the emitted row against the first qualifying operation in canonical history order for that ledger.
4. The command can export the tx-set-first asset instead of the history-first asset.

## Target Code

- `internal/input/assets.go:GetPaymentOperations:38-56` — iterates `ledger.TransactionEnvelopes()` directly and applies `limit` against that order
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:28-45` — iterates `transform.GetTransactionSet(ledger)` directly and applies the same limit logic
- `internal/input/transactions.go:GetTransactions:43-62` — sibling reader that uses `LedgerTransactionReader.Read()` for processed transaction order
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_reader.go:123-131` — SDK comment states tx-set envelopes are not in the same order as processed metas
- `cmd/export_assets.go:30-45` — `export_assets` takes the reader output as-is before deduplication and row limiting semantics are exposed to users

## Evidence

The asset readers are the only live transaction-scanning readers in this subsystem that do not normalize through `LedgerTransactionReader`. The SDK explicitly documents why that normalization exists: raw envelopes come from the agreed transaction set, while the actual per-transaction metadata is ordered differently. Because `export_assets` exposes `--limit` at the reader-output level, this ordering difference can change which asset rows are emitted, not just their presentation order.

## Anti-Evidence

If `export_assets` were explicitly documented as a raw tx-set scan rather than a history-order export, selecting rows in tx-set order would be defensible. The current command surface only documents a maximum number of exported objects over a ledger range, and sibling transaction-based exports normalize to processed history order, so the raw-order choice still looks like a silent data-selection bug.
