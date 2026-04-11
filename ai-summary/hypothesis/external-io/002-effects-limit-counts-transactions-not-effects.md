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
