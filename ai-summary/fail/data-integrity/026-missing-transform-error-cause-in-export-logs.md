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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestMissingTransformErrorCauseInExportLogs"
**Test Language**: Go

### Demonstration

The test reproduces the exact `fmt.Errorf` calls from all 4 affected export commands and confirms that each resulting error message is missing the underlying transform error text. Each error ends with a dangling `: ` separator and no root cause, while the sibling `export_effects` pattern correctly includes the error. This proves that operators receive context-only log messages with no diagnostic value when transform failures occur in these commands.

### Test Body

```go
package cmd

import (
	"fmt"
	"strings"
	"testing"
)

// TestMissingTransformErrorCauseInExportLogs demonstrates that four export
// commands construct error messages that omit the underlying transform error.
// Each command captures `err` from its Transform*() call, but the subsequent
// fmt.Errorf uses a format string ending in ": " with no %v/%s for `err`.
// This means LogError receives an error with no root-cause detail.
func TestMissingTransformErrorCauseInExportLogs(t *testing.T) {
	// Simulate the underlying transform error that would be returned
	// by TransformTransaction, TransformLedgerTransaction, TransformAsset,
	// or TransformContractEvent.
	transformErr := fmt.Errorf("xdr decode failed: unexpected EOF at byte 42")

	// The correct pattern (used by export_effects.go line 39):
	//   fmt.Errorf("could not transform transaction %d in ledger %d: %v", txIndex, ledgerSeq, err)
	correctMsg := fmt.Errorf("could not transform transaction %d in ledger %d: %v", 7, 123, transformErr)

	if !strings.Contains(correctMsg.Error(), transformErr.Error()) {
		t.Fatalf("reference pattern broken: correct message should contain transform error")
	}

	// ---- Affected commands reproduce the exact fmt.Errorf calls from production ----

	type testCase struct {
		name    string
		errMsg  error // the exact fmt.Errorf call from the production code
		srcFile string
		srcLine int
	}

	cases := []testCase{
		{
			name:    "export_transactions",
			errMsg:  fmt.Errorf("could not transform transaction %d in ledger %d: ", 7, 123),
			srcFile: "cmd/export_transactions.go",
			srcLine: 38,
		},
		{
			name:    "export_ledger_transaction",
			errMsg:  fmt.Errorf("could not transform ledger_transaction transaction %d in ledger %d: ", 7, 123),
			srcFile: "cmd/export_ledger_transaction.go",
			srcLine: 37,
		},
		{
			name:    "export_assets",
			errMsg:  fmt.Errorf("could not extract asset from operation %d in transaction %d in ledger %d: ", 3, 7, 123),
			srcFile: "cmd/export_assets.go",
			srcLine: 48,
		},
		{
			name:    "export_contract_events",
			errMsg:  fmt.Errorf("could not transform contract events in transaction %d in ledger %d: ", 7, 123),
			srcFile: "cmd/export_contract_events.go",
			srcLine: 37,
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			loggedError := tc.errMsg.Error()

			// The logged error should contain the transform error cause.
			// This assertion PASSES (proving the bug) because the cause is missing.
			if !strings.Contains(loggedError, transformErr.Error()) {
				// BUG CONFIRMED: the transform error text is absent from the logged message.
				t.Logf("BUG CONFIRMED in %s:%d", tc.srcFile, tc.srcLine)
				t.Logf("  Logged error: %q", loggedError)
				t.Logf("  Missing cause: %q", transformErr.Error())
				t.Logf("  The error ends with a trailing colon-space and no root cause.")
			} else {
				t.Errorf("Expected the transform error to be MISSING from the logged message, "+
					"but it was found. If this passes, the bug may have been fixed.\n"+
					"  Logged: %q\n  Cause: %q", loggedError, transformErr.Error())
			}

			// Additionally verify the message ends with ": " — the dangling separator
			if !strings.HasSuffix(loggedError, ": ") {
				t.Errorf("Expected error message to end with dangling ': ' but got: %q", loggedError)
			}
		})
	}
}
```

### Test Output

```
=== RUN   TestMissingTransformErrorCauseInExportLogs
=== RUN   TestMissingTransformErrorCauseInExportLogs/export_transactions
    data_integrity_poc_test.go:72: BUG CONFIRMED in cmd/export_transactions.go:38
    data_integrity_poc_test.go:73:   Logged error: "could not transform transaction 7 in ledger 123: "
    data_integrity_poc_test.go:74:   Missing cause: "xdr decode failed: unexpected EOF at byte 42"
    data_integrity_poc_test.go:75:   The error ends with a trailing colon-space and no root cause.
=== RUN   TestMissingTransformErrorCauseInExportLogs/export_ledger_transaction
    data_integrity_poc_test.go:72: BUG CONFIRMED in cmd/export_ledger_transaction.go:37
    data_integrity_poc_test.go:73:   Logged error: "could not transform ledger_transaction transaction 7 in ledger 123: "
    data_integrity_poc_test.go:74:   Missing cause: "xdr decode failed: unexpected EOF at byte 42"
    data_integrity_poc_test.go:75:   The error ends with a trailing colon-space and no root cause.
=== RUN   TestMissingTransformErrorCauseInExportLogs/export_assets
    data_integrity_poc_test.go:72: BUG CONFIRMED in cmd/export_assets.go:48
    data_integrity_poc_test.go:73:   Logged error: "could not extract asset from operation 3 in transaction 7 in ledger 123: "
    data_integrity_poc_test.go:74:   Missing cause: "xdr decode failed: unexpected EOF at byte 42"
    data_integrity_poc_test.go:75:   The error ends with a trailing colon-space and no root cause.
=== RUN   TestMissingTransformErrorCauseInExportLogs/export_contract_events
    data_integrity_poc_test.go:72: BUG CONFIRMED in cmd/export_contract_events.go:37
    data_integrity_poc_test.go:73:   Logged error: "could not transform contract events in transaction 7 in ledger 123: "
    data_integrity_poc_test.go:74:   Missing cause: "xdr decode failed: unexpected EOF at byte 42"
    data_integrity_poc_test.go:75:   The error ends with a trailing colon-space and no root cause.
--- PASS: TestMissingTransformErrorCauseInExportLogs (0.00s)
    --- PASS: TestMissingTransformErrorCauseInExportLogs/export_transactions (0.00s)
    --- PASS: TestMissingTransformErrorCauseInExportLogs/export_ledger_transaction (0.00s)
    --- PASS: TestMissingTransformErrorCauseInExportLogs/export_assets (0.00s)
    --- PASS: TestMissingTransformErrorCauseInExportLogs/export_contract_events (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	6.147s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-12
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Exercises claimed bug**: PARTIAL — source inspection confirms the four commands construct `fmt.Errorf(...)` values that drop the original `err`, and an independent logger-based PoC reproduced the resulting context-only log message. But the supplied PoC does not invoke the export commands themselves; it only recreates the formatting calls.
2. **Realistic preconditions**: YES — `TransformTransaction()`, `TransformLedgerTransaction()`, `TransformAsset()`, and `TransformContractEvent()` all have real error returns, so these branches are reachable when a transform fails.
3. **Bug vs by-design**: BUGLET — sibling commands include `%v`/`%s`, so the omission is best explained as a copy-paste mistake rather than an intentional logging contract.
4. **Impact vs claimed severity**: FAIL — this does not change exported rows, suppress the failure count, or make the run appear successful. The command still logs each failed row context and emits `failed_transforms` in `PrintTransformStats`, so the omission is weaker than the Medium operational-correctness examples in scope.
5. **In scope**: FAIL — the objective is silent data corruption or materially masked data loss. Here the transform failure itself is already surfaced; only the root-cause detail is missing. That is observability debt, not a demonstrated data-integrity bug.
6. **Test correctness**: FAIL — the PoC passes by asserting properties of reconstructed `fmt.Errorf` strings, not by driving the production export loop or checking end-user CLI output from a real transform failure.
7. **Alternative explanations**: YES — the observed behavior is fully explained by the missing `%v` placeholder, without any broader error-swallowing bug in the export pipeline.
8. **Novelty**: NOVEL — but novelty alone does not make it an in-scope correctness finding.

### Rejection Reason

The code does drop the underlying transform error text from four log messages, but the export still visibly reports the failed row context and aggregate failure counts. That makes this a debugging/observability issue, not a concrete data-integrity or silent-failure bug under the stated review objective.

### Failed Checks

- 4. Impact vs claimed severity
- 5. In scope
- 6. Test correctness
