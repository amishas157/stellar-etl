# H002: Datastore-backed `export_assets` accepts impossible ledger ranges and emits empty success

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Structural data corruption: impossible asset ranges can be exported as plausible empty files
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a bounded asset export is impossible — for example `end == 0` or `end < start` — the command should fail before producing output. The repository already has a shared `ValidateLedgerRange()` helper for this exact contract, and a bounded export should not silently translate invalid caller input into “no assets existed in this range.”

## Mechanism

The default datastore path in `export_assets` goes through `GetPaymentOperations()`, which uses `CreateLedgerBackend()` instead of the validating history-archive helper. That backend path never calls `ValidateLedgerRange()`, and the reader then loops `for seq := start; seq <= end; seq++`; impossible ranges simply skip the loop and return an empty slice with no error. `cmd/export_assets` treats that empty slice as a successful export and can even upload the empty artifact, creating a success-shaped but false “no rows” result.

## Trigger

1. Run the default datastore-backed command without `--captive-core`, for example:
   - `stellar-etl export_assets --start-ledger 100 --end-ledger 50 -o /tmp/assets.json`
   - `stellar-etl export_assets --start-ledger 100 --end-ledger 0 -o /tmp/assets.json`
2. Inspect the command result and output file.
3. The command should fail validation, but the current path can succeed with an empty asset file and normal success stats.

## Target Code

- `cmd/export_assets.go:Run:30-36` — default path chooses `input.GetPaymentOperations(...)` whenever `--captive-core` is not set.
- `internal/input/assets.go:GetPaymentOperations:21-60` — prepares a bounded range and then iterates `for seq := start; seq <= end; seq++` with no explicit range validation.
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — datastore/captive-core backend creation path does not call `ValidateLedgerRange()`.
- `internal/utils/main.go:ValidateLedgerRange:736-758` — existing shared validator that would reject `end == 0` and `end < start`.
- `internal/utils/main.go:CreateBackend:760-778` — contrasting history-archive path that does validate bounded ranges.

## Evidence

This is the same empty-success control-flow shape already confirmed for the transaction-family and ledger-family readers, but `export_assets` is not covered by those findings. The split between `CreateLedgerBackend()` (no validation) and `CreateBackend()` (calls `ValidateLedgerRange()`) makes the bug backend-specific and reachable in the command’s default path today.

## Anti-Evidence

The `--captive-core` branch of `export_assets` uses `GetPaymentOperationsHistoryArchive()`, which reaches `CreateBackend()` and therefore rejects impossible bounded ranges correctly. A reviewer could argue this limits the blast radius, but the default datastore path is the ordinary production configuration, so the silent empty-success behavior is still live for normal users.
