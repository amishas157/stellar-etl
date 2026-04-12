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
