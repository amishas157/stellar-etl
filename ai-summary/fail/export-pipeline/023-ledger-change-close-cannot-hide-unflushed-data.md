# H023: Ignored `LedgerChangeReader.Close()` errors can hide unflushed batch data

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If closing a per-ledger change reader is required to flush buffered data or
surface a deferred backend error, `extractBatch()` should check the returned
error and fail the batch instead of reporting a complete export.

## Mechanism

`extractBatch()` calls `changeReader.Close()` and discards the returned error. I
initially suspected that a close-time failure could hide unread or unflushed
ledger changes, causing a batch to look complete even though the underlying
reader reported a problem on shutdown.

## Trigger

Run `export_ledger_entry_changes` on a backend / reader implementation whose
`Close()` surfaces a deferred read failure, transport failure, or buffer-flush
error only at shutdown.

## Target Code

- `internal/input/changes.go:138-139` — `extractBatch()` calls `changeReader.Close()` and ignores the result
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_change_reader.go:207-210` — `LedgerChangeReader.Close()` just clears pending state and delegates to `LedgerTransactionReader.Close()`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledger_transaction_reader.go:152-157` — `LedgerTransactionReader.Close()` only nils an in-memory map and returns `nil`

## Evidence

The export path really does ignore the close result, so if shutdown were the
place where the SDK reported deferred problems, this would be a silent-success
bug. The pattern is also inconsistent with the rest of the function, which
fatally aborts on read-time errors.

## Anti-Evidence

In the current SDK implementation, `LedgerChangeReader.Close()` does not perform
I/O, flushing, or validation. It simply drops in-memory buffers and calls
`LedgerTransactionReader.Close()`, which itself only sets `envelopesByHash = nil`
and returns `nil`.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Failed At**: hypothesis
**Novelty**: PASS — not previously investigated

### Why It Failed

The ignored `Close()` result cannot currently hide data loss because the concrete
SDK close methods do not report deferred backend errors or flush pending export
data. Today the call is effectively side-effect cleanup with a hardcoded `nil`
result.

### Lesson Learned

Unchecked close calls are only meaningful integrity bugs when the underlying
implementation does real finalization work. Before escalating an ignored close
error, inspect the concrete SDK methods rather than assuming they behave like
buffered writers or network streams.
