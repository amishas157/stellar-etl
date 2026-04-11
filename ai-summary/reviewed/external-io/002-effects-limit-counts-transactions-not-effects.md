# H002: `export_effects --limit` caps transactions, not emitted effect rows

**Date**: 2026-04-11
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For `export_effects`, the shared `--limit` flag is registered as "Maximum number of effects to export", so a run with `--limit N` should write at most `N` effect rows to JSON/Parquet output.

## Mechanism

`cmd/export_effects.go` passes that limit into `input.GetTransactions()`, which stops after `N` transactions, not `N` effects. `transform.TransformEffect()` then expands each returned transaction into `[]EffectOutput`, and `export_effects` writes every element of that slice, so a single transaction with multiple effects can exceed the promised row limit and silently export extra rows.

## Trigger

Run `export_effects --limit 1` on a ledger containing a successful transaction that produces multiple effects, such as account-creation or path-payment flows that touch several ledger entries. The command should emit one effect row but should instead write all effects from that single transaction.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:248-255` - CLI help text promises a maximum number of `effects`
- `cmd/export_effects.go:25-69` - forwards the effects limit into `GetTransactions()` and then writes every transformed effect
- `internal/input/transactions.go:GetTransactions:22-70` - enforces `limit` on transaction count
- `internal/transform/effects.go:23-50` - expands one transaction into a slice of effect rows

## Evidence

The effect command's own inline comment says `limit: maximum number of effects to export`, but the actual reader boundary is transaction-granular. `TransformEffect()` explicitly appends `p...` from every operation in the transaction, so one limited transaction can still produce many emitted rows.

## Anti-Evidence

If each returned transaction happens to yield exactly one effect, the mismatch is invisible. Negative limits also avoid the boundary entirely because the command exports the whole range.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`export_effects.go:25` calls `input.GetTransactions(startNum, commonArgs.EndNum, limit, ...)`, passing the user-supplied `--limit` value. `GetTransactions()` (transactions.go:51) enforces the limit against `len(txSlice)` — the number of *transactions*, not effects. Back in `export_effects.go:34-57`, every transaction is expanded via `TransformEffect()` into a `[]EffectOutput` slice, and every element is written to output with no secondary count check. The sibling commands `export_operations` and `export_trades` each have dedicated `Get*()` functions that count the actual output entity (operations, trade-producing operations) against the limit, confirming that the effects path is the outlier.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:250-254` — Registers `--limit` with help text `"Maximum number of " + objectName + " to export"`. For effects, objectName="effects", so the flag promises to limit effects.
- `cmd/export_effects.go:25` — Passes `limit` directly to `input.GetTransactions()`, which treats it as a transaction cap.
- `internal/input/transactions.go:51` — Loop condition `for int64(len(txSlice)) < limit || limit < 0` counts transactions against limit.
- `cmd/export_effects.go:34-57` — Iterates all returned transactions, calls `TransformEffect()` per transaction, then writes every effect in the returned slice. No limit enforcement on the emitted effect count.
- `internal/transform/effects.go:23-50` — `TransformEffect()` iterates all operations in a transaction and appends all effects from each, returning a potentially large `[]EffectOutput` per transaction.
- `internal/input/operations.go:52-70` — Sibling `GetOperations()` correctly counts *operations* against the limit with an inner break, confirming the expected pattern.
- `internal/input/trades.go:50-76` — Sibling `GetTrades()` correctly counts *trade inputs* against the limit with an inner break.

### Findings

The bug is confirmed. The `--limit N` flag for `export_effects` promises to cap output at N effects but actually caps at N transactions. Since a single transaction can produce many effects (every operation in a successful transaction generates one or more effects — account creation alone produces 3 effects: account_created, signer_created, account_debited), the actual output row count can significantly exceed the requested limit.

**Severity downgrade from High to Medium**: The individual effect rows are all correct — no data corruption occurs. The issue is that the limit control doesn't work as documented, which is an operational correctness problem rather than structural data corruption. Output consumers receive more rows than expected, but each row contains accurate data.

The pattern is consistent with Investigation Pattern 5 (Loop condition consistency): `GetOperations()` and `GetTrades()` each count their actual output entity against the limit, while `export_effects` reuses `GetTransactions()` without adaptation, making it the outlier.

### PoC Guidance

- **Test file**: `cmd/export_effects_test.go` (or a new test file if one doesn't exist)
- **Setup**: Create a mock ledger backend containing a single transaction that produces multiple effects (e.g., a `CreateAccount` operation which generates `account_created`, `signer_created`, and `account_debited` effects). Configure `--limit 1`.
- **Steps**: Call the effects export path with `limit=1`. Count the number of effect rows written to the output file.
- **Assertion**: Assert that the output contains more than 1 effect row despite `--limit 1`, demonstrating the limit is enforced on transactions, not effects. Alternatively, a unit-level PoC: call `GetTransactions(start, end, 1, ...)` and then `TransformEffect()` on the returned transaction, and assert `len(effects) > 1`.
