# H004: Several export commands drop the underlying transform error from partial-export logs

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a live export command skips a row because the transform failed, the logged
error should include the underlying `err` so operators can identify the bad code
path and distinguish one-off row issues from systemic export corruption. In
strict mode, the fatal error should surface the same root cause rather than a
context-only message.

## Mechanism

Several commands call `cmdLogger.LogError(fmt.Errorf("..."))` without actually
formatting the original transform error into the message. The row is counted as
failed and skipped, but the operator only sees a truncated context string such
as `could not transform transaction 7 in ledger 123:` with no cause attached.
That makes partial exports materially harder to diagnose, and in `--strict-export`
mode it can abort the run without revealing which transform bug or malformed row
triggered the stop.

## Trigger

1. Run any of the affected commands over a ledger range that hits a real
   transform failure.
2. Examples include a transaction row that makes `TransformTransaction()`
   return an error, a contract-event row that makes `TransformContractEvent()`
   fail, or an asset row that makes `TransformAsset()` fail.
3. Inspect the logged error: the command reports the row context, but the
   underlying transform error text is missing from the message.

## Target Code

- `cmd/export_transactions.go:35-39` — logs `could not transform transaction ...`
  without appending `err`
- `cmd/export_ledger_transaction.go:34-38` — same missing-error pattern for
  `TransformLedgerTransaction()`
- `cmd/export_assets.go:45-49` — same missing-error pattern for `TransformAsset()`
- `cmd/export_contract_events.go:34-38` — same missing-error pattern for
  `TransformContractEvent()`
- `cmd/export_effects.go:39` — sibling command includes `%v`, showing the
  intended error-reporting pattern

## Evidence

The affected commands all increment their failure counts and continue, so these
branches are live in normal partial-export operation. But unlike the sibling
`export_effects` path, the format strings omit `%v` or error wrapping, which
means `LogError` receives a context-only error object with the original cause
already discarded.

## Anti-Evidence

The commands do still record a failure and expose ledger / transaction / operation
coordinates, so this is not a fully silent corruption path. Review should confirm
whether the project treats missing root-cause text in partial-export logs as an
operational correctness bug or merely a debugging-quality issue.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced all `LogError(fmt.Errorf(...))` calls across every `cmd/export_*.go` file. Exactly 4 commands construct an error message with a trailing `: ` but never append the `err` variable via `%v` or `%s`. The remaining 6+ sibling commands consistently include the error. The `LogError` function either calls `l.Fatal(err)` (strict mode) or `l.Error(err)` (normal mode), so the truncated message is the only diagnostic output an operator receives.

### Code Paths Examined

- `cmd/export_transactions.go:38` — `fmt.Errorf("could not transform transaction %d in ledger %d: ", ...)` — format string ends with `: ` and no `%v`/`err`; the `err` from `TransformTransaction()` on line 36 is captured but never used
- `cmd/export_ledger_transaction.go:37` — identical pattern: `fmt.Errorf("could not transform ledger_transaction transaction %d in ledger %d: ", ...)` — `err` from line 34 discarded
- `cmd/export_assets.go:48` — `fmt.Errorf("could not extract asset from operation %d in transaction %d in ledger %d: ", ...)` — `err` from line 45 discarded
- `cmd/export_contract_events.go:37` — `fmt.Errorf("could not transform contract events in transaction %d in ledger %d: ", ...)` — `err` from line 34 discarded
- `cmd/export_effects.go:39` — `fmt.Errorf("could not transform transaction %d in ledger %d: %v", txIndex, LedgerSeq, err)` — correct pattern, includes `err`
- `cmd/export_operations.go:38` — correct pattern, includes `err`
- `cmd/export_ledgers.go:45` — correct pattern, includes `err`
- `cmd/export_trades.go:41` — correct pattern, includes `err`
- `cmd/export_token_transfers.go:41` — correct pattern, includes `err`
- `internal/utils/logger.go:17-23` — `LogError` either fatals or errors with the passed error; no fallback error enrichment

### Findings

1. **The bug is real and consistent across 4 commands.** Each affected `fmt.Errorf` call ends with `: ` — a clear trailing colon-space that was intended to precede `%v` + `err`. This is a copy-paste omission.
2. **Sibling commands prove the intended pattern.** 6+ other export commands in the same directory use `%v` or `%s` with `err` in the identical position, confirming this was not a deliberate omission.
3. **`--strict-export` amplifies the impact.** In strict mode, `LogError` calls `Fatal(err)`, which aborts the entire export run. The abort message will contain only the context string (e.g., `could not transform transaction 7 in ledger 123: `) with no root cause, making it impossible to distinguish between XDR deserialization failures, type assertion failures, missing fields, or other transform errors.
4. **Normal mode loses diagnostic value.** Without strict mode, the error is logged and the row is skipped. Operators monitoring logs for partial-export quality cannot identify the nature of failures or determine if they are systemic.
5. **Note on `export_contract_events`:** This command also has a separately confirmed bug where `--strict-export` is never wired to `cmdLogger.StrictExport` (see `success/cli-commands/009`). Even if the missing-error bug were fixed, `export_contract_events` would still not abort on transform errors in strict mode due to the unwired flag.

### PoC Guidance

- **Test file**: `cmd/export_transactions_test.go` (or a new test helper)
- **Setup**: Construct a `TransformTransactionInput` that will cause `TransformTransaction()` to return a non-nil error (e.g., an input with a corrupted or empty `LedgerHistory`)
- **Steps**: Run the transaction export loop over the crafted input and capture the error message passed to `LogError`
- **Assertion**: Assert that the logged error message contains the original `err.Error()` text, not just the context prefix. Currently it will only contain `"could not transform transaction N in ledger M: "` with no error detail after the colon.
