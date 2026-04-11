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
