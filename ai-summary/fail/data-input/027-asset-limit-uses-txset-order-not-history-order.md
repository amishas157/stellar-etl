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

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the complete path from `GetPaymentOperations()` through `TransactionEnvelopes()` (tx-set order) and compared against `GetTransactions()` which uses `LedgerTransactionReader.Read()` (processing/hash-sorted order). Confirmed the SDK comment at `ledger_transaction_reader.go:126-130` that these two orderings can differ. However, traced `TransformAsset()` and the `AssetOutput` schema to determine whether any ordering-dependent field is actually exported — none is. The `operationID` computed from the tx-set-order `TransactionIndex` is used solely in error messages and never appears in the output struct.

### Code Paths Examined

- `internal/input/assets.go:GetPaymentOperations:38-56` — confirmed iteration over `TransactionEnvelopes()` (tx-set order), `TransactionIndex` is `int32(txIndex)` from that order
- `internal/input/assets_history_archive.go:GetPaymentOperationsHistoryArchive:28-44` — same pattern, uses `transform.GetTransactionSet()` which also returns tx-set order
- `internal/input/transactions.go:GetTransactions:43-62` — confirmed uses `LedgerTransactionReader.Read()` which yields processing order
- `go-stellar-sdk/ingest/ledger_transaction_reader.go:76-106` — `Read()` iterates `readIdx` over `TxProcessing` (processing order), resolves envelope by hash lookup
- `go-stellar-sdk/ingest/ledger_transaction_reader.go:125-147` — `storeTransactions()` comment explicitly states tx-set and meta orderings differ
- `go-stellar-sdk/xdr/ledger_close_meta.go:62-94` — `TransactionEnvelopes()` returns from TxSet phases (tx-set order)
- `internal/transform/asset.go:14-53` — `TransformAsset()` computes `operationID` from `transactionIndex` but uses it ONLY in error messages (lines 19, 27, 34, 42), never stores it in `AssetOutput`
- `internal/transform/schema.go:227-235` — `AssetOutput` struct contains only `AssetCode`, `AssetIssuer`, `AssetType`, `AssetID` (deterministic farm hash), `ClosedAt`, `LedgerSequence` — NO ordering-dependent fields

### Why It Failed

The hypothesis claims `export_assets --limit` produces "wrong data" due to tx-set order vs processing order. However, no output field in `AssetOutput` is ordering-dependent:

1. **No ordering field is exported**: The `operationID` computed from `TransactionIndex` is only used in error messages — it never enters the output schema. `AssetOutput` contains only asset code/issuer/type, a deterministic hash ID, close time, and ledger sequence.

2. **Each individual asset row is correct**: Regardless of scan order, any asset extracted from a payment/manage-sell operation is correctly extracted. The `AssetID` is a deterministic farm hash of `(code, issuer, type)`, independent of scan order.

3. **The "wrong subset" effect is philosophical, not a data corruption bug**: With `--limit`, a different scan order might examine different operations first, potentially discovering different unique assets. But each discovered asset is valid — getting asset A instead of asset B when both are legitimate payment-derived assets is not "wrong data."

4. **Previous reviews characterized this as by-design**: Fail 017 explicitly noted `export_assets` is "an envelope-derived asset-reference export over payment operations." The tx-set-order scan is the intentional design, not a deviation from expected behavior.

5. **The `--limit` parameter caps operations scanned, not output ordering**: The flag comment says "maximum number of operations to export." It does not promise a canonical history-ordered prefix.

### Lesson Learned

When a hypothesis claims ordering produces "wrong data," verify whether any ordering-dependent field actually appears in the output schema. If the output contains only content-derived fields (asset code/issuer/type) and deterministic IDs (farm hash), scan order affects which subset is examined under `--limit` but does not make any individual row incorrect. The `export_assets` command is intentionally envelope-derived — previous reviews (fail 017) established this design intent.
