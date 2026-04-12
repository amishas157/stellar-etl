# H002: `export_token_transfer --limit` caps ledgers instead of exported token-transfer rows

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural row-count corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a user requests `export_token_transfer --limit 1`, the command should emit at most one token-transfer row, because the shared archive flag is documented as the maximum number of exported `token_transfer` objects. A caller using the flag for batch sizing should not receive dozens of extra rows from a single successful run.

## Mechanism

`export_token_transfer` passes the user limit into `input.GetLedgers()`, and `GetLedgers()` enforces that limit as "maximum ledgers read," not "maximum token-transfer rows exported." `transform.TransformTokenTransfer()` then expands each returned ledger into a slice of token-transfer events, and the command writes every event from that ledger without any second-stage row cap.

That means `--limit 1` really means "read one ledger," even though the CLI flag advertises a token-transfer limit. On ledgers with many Soroban token events, the JSON output silently contains far more rows than requested while the command reports normal success.

## Trigger

Run:

`stellar-etl export_token_transfer -s 30820015 -e 30820015 -l 1 -o /tmp/token-transfers.json`

The existing one-ledger fixture for this exact ledger (`testdata/token_transfers/one_ledger_token_transfers.golden`) contains 60 token-transfer rows, so a user-requested limit of 1 still produces a 60-row export.

## Target Code

- `internal/utils/main.go:AddArchiveFlags:250-254` — public flag contract says `limit` is the maximum number of exported `token_transfer` objects
- `cmd/export_token_transfers.go:21-29` — forwards the user limit into `GetLedgers()`
- `cmd/export_token_transfers.go:38-54` — writes every token-transfer row returned for each ledger
- `internal/input/ledgers.go:GetLedgers:13-90` — enforces `limit` on ledgers, not transformed token-transfer rows
- `internal/transform/token_transfer.go:14-35` — expands one ledger into a slice of token-transfer outputs
- `internal/transform/token_transfer.go:37-129` — appends one exported row per token-transfer event

## Evidence

`GetLedgers()` breaks after `len(ledgerSlice) >= limit`, so `-l 1` guarantees exactly one ledger input. But `TransformTokenTransfer()` returns `[]TokenTransferOutput`, and the checked-in fixture for ledger `30820015` contains 60 rows. The command then loops over that slice and writes every row, proving the effective limit is on ledgers rather than the exported object type named by the flag.

## Anti-Evidence

One could argue that token-transfer export is ledger-driven and therefore a ledger limit is convenient internally. But that interpretation contradicts the shared archive flag text that every CLI user sees, and sibling single-row-per-input commands do make `limit` line up with exported object count, so this command's behavior is a misleading outlier.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated for token_transfer, but substantially equivalent to fail/025 (assets limit overrun)
**Failed At**: reviewer

### Trace Summary

Traced the limit enforcement path from `AddArchiveFlags("token_transfer", ...)` through `GetLedgers()` (ledgers.go:85) and the export loop in `export_token_transfers.go:38-54`. Confirmed the limit is applied at the ledger level, not the token-transfer row level. Then compared this pattern against ALL sibling commands and found it is NOT an outlier — `export_effects` has the identical disconnect (flag says "Maximum number of effects" but `GetTransactions` limits transactions, not effects). The command's own init comment (line 77) explicitly documents `limit: maximum number of ledgers to export`.

### Code Paths Examined

- `internal/utils/main.go:AddArchiveFlags:250-254` — auto-generates flag text "Maximum number of token_transfer to export"
- `cmd/export_token_transfers.go:21-29` — passes limit to `GetLedgers()`, which enforces it on ledger count
- `cmd/export_token_transfers.go:38-54` — iterates all token transfers from each ledger, no second-stage row cap
- `cmd/export_token_transfers.go:77` — init comment explicitly says "limit: maximum number of ledgers to export; default to 60"
- `internal/input/ledgers.go:85` — `if int64(len(ledgerSlice)) >= limit && limit >= 0 { break }` — limit on ledgers
- `internal/input/transactions.go:51` — `GetTransactions` limits transactions, used by `export_effects` which claims to limit effects
- `cmd/export_effects.go:25` — effects uses `GetTransactions()` with same granularity mismatch
- `internal/input/operations.go:52,67` — operations enforces limit per-operation (finer granularity)
- `internal/input/trades.go:50,73` — trades enforces limit per-trade-input (finer granularity)

### Why It Failed

This is the same pattern already rejected in fail/025 (`export_assets --limit` overruns). The reasoning applies identically:

1. **All output row values are correct.** Every token-transfer row contains the correct from, to, amount, asset, ledger_sequence, etc. No field holds a wrong value.
2. **The behavior is not an outlier.** `export_effects` has the identical granularity mismatch — its flag says "Maximum number of effects" but `GetTransactions` limits transactions, not effects. The hypothesis's claim that "sibling single-row-per-input commands make limit line up" is only true for `export_transactions` and `export_operations`, not for all multi-row-expansion commands.
3. **The init comment documents the actual behavior.** Line 77 of `export_token_transfers.go` explicitly says "limit: maximum number of ledgers to export" — the developer understood and intended this.
4. **Extra correct data is not data corruption.** Downstream systems (BigQuery, analytics) process variable row counts. Receiving 60 correct token-transfer rows instead of 1 does not cause wrong calculations, hash collisions, or compliance violations.
5. **This is a flag help text imprecision, not a data integrity bug.** `AddArchiveFlags` auto-generates the flag description using the object name, creating misleading text for any command where the input granularity (ledgers, transactions) differs from the output granularity (token transfers, effects). This is a documentation concern, not silent data corruption.

### Lesson Learned

The `AddArchiveFlags` auto-generated limit text creates a misleading contract for ALL multi-row-expansion commands (`export_token_transfer`, `export_effects`, `export_assets`). This is a systemic documentation issue, not a per-command data correctness bug. When investigating limit semantics, check the command's init comment block for the developer's stated intent, and verify whether the pattern is actually unique before claiming "outlier."
