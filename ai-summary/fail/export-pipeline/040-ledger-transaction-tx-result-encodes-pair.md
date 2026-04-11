# H002: `export_ledger_transaction` serializes `tx_result` as `TransactionResultPair`

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`tx_result` should export the XDR `TransactionResult` body for the transaction, matching the field name and the sibling `history_transactions.tx_result` export. A consumer decoding the blob as `xdr.TransactionResult` should recover the same fee-charged and result-code payload that `TransformTransaction()` exports for the same transaction.

## Mechanism

`ingest.LedgerTransaction.Result` is an `xdr.TransactionResultPair`, not a plain `xdr.TransactionResult`. `TransformLedgerTransaction()` marshals `&transaction.Result`, so every `tx_result` blob is prefixed with the transaction hash and encoded as the pair type instead of the result body, which makes the raw export structurally diverge from the rest of the pipeline while still looking like plausible base64.

## Trigger

Run `export_ledger_transaction` over any non-empty ledger range, then decode one emitted `tx_result` blob as `xdr.TransactionResult` or compare it with `TransformTransaction(...).TxResult` for the same transaction. The ledger-transaction export will contain the pair-encoded payload instead of the plain result body.

## Target Code

- `internal/transform/ledger_transaction.go:22-25` — marshals `&transaction.Result`
- `internal/transform/transaction.go:54-57` — sibling transaction export marshals `&transaction.Result.Result`
- `internal/transform/schema.go:86-94` — field is named singular `tx_result`, not `tx_result_pair`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20260220172402-fa15bc76ef5d/ingest/ledger_transaction.go:16-20` — `LedgerTransaction.Result` is typed as `xdr.TransactionResultPair`

## Evidence

The two transaction exporters serialize different XDR objects for fields with the same semantic name. `TransformTransaction()` explicitly drills into `.Result.Result`, while `TransformLedgerTransaction()` stops at `.Result`, which is the pair wrapper that also embeds the transaction hash.

## Anti-Evidence

The existing `ledger_transaction` tests currently expect pair-encoded blobs, and the command is intentionally lower-level than the parsed transaction export. If maintainers wanted this command to expose the raw result pair despite the singular field name, the mismatch may be intentional rather than accidental.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced `TransformLedgerTransaction()` in `internal/transform/ledger_transaction.go` which serializes each field of the `ingest.LedgerTransaction` struct at its natural level: `transaction.Envelope` (TransactionEnvelope), `&transaction.Result` (TransactionResultPair), `transaction.UnsafeMeta` (TransactionMeta), `transaction.FeeChanges` (LedgerEntryChanges). This is a consistent raw-dump pattern. Critically, `LedgerTransactionOutput` has no separate `tx_hash` column — the transaction hash is only recoverable from the `TransactionResultPair` blob. Stripping to `TransactionResult` would lose the hash entirely, making the export less useful. The test file (`ledger_transaction_test.go:49-68`) explicitly constructs `TransactionResultPair` inputs with known hashes and asserts the pair-encoded base64 output, confirming this is tested and intentional.

### Code Paths Examined

- `internal/transform/ledger_transaction.go:22` — `xdr.MarshalBase64(&transaction.Result)` serializes the full `TransactionResultPair`
- `internal/transform/ledger_transaction.go:17,27,32` — sibling fields also serialize raw struct members (Envelope, UnsafeMeta, FeeChanges)
- `internal/transform/schema.go:86-94` — `LedgerTransactionOutput` has no `tx_hash` field; hash is only in the result pair
- `internal/transform/ledger_transaction_test.go:144-153,198-223,267-276` — tests construct `TransactionResultPair` inputs with explicit hashes
- `internal/transform/ledger_transaction_test.go:49-68` — expected outputs contain pair-encoded base64 (first 32 bytes decode to the transaction hash)
- `internal/transform/transaction.go:54` — `TransformTransaction()` uses `.Result.Result` because it extracts hash separately at line 22
- `xdr/xdr_generated.go:14891-14894` — `TransactionResultPair` = `{ TransactionHash Hash; Result TransactionResult }`

### Why It Failed

Working as designed. The `export_ledger_transaction` command is a raw XDR dump that serializes each `LedgerTransaction` struct field at its native level. The pair encoding is necessary because `LedgerTransactionOutput` has no separate `tx_hash` column — the transaction hash is embedded only in the `TransactionResultPair`. Stripping to `TransactionResult` would make transactions unidentifiable. The sibling `TransformTransaction()` drills into `.Result.Result` because it exports the hash separately via `utils.HashToHexString(transaction.Result.TransactionHash)` at line 22. The divergence between the two exporters reflects their different design intents (raw blob dump vs parsed fields), not a copy-paste omission. All three test cases explicitly assert pair-encoded output.

### Lesson Learned

When two export functions serialize the same underlying data at different levels of nesting, check whether the higher-level export extracts the "missing" data (e.g., hash) into a separate column. If the raw export has no such column, the deeper nesting level is likely intentional to preserve information that would otherwise be lost.
