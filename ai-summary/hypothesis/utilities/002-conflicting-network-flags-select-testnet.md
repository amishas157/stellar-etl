# H002: Conflicting network flags silently export testnet data

**Date**: 2026-04-10
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a user sets both `--testnet` and `--futurenet`, the ETL should reject the invocation as an invalid configuration instead of exporting data from one network arbitrarily. Exported JSON/Parquet should always come from the network the CLI invocation unambiguously selected.

## Mechanism

`MustCommonFlags` and `get_ledger_range_from_times` read both booleans independently and never enforce mutual exclusion. `GetEnvironmentDetails` then resolves the ambiguity with `if commonFlags.IsTest { ... } else if commonFlags.IsFuture { ... }`, which means `--testnet --futurenet` always selects testnet archives, passphrase, and core config. The CLI therefore emits a structurally valid dataset from the wrong network without warning.

## Trigger

Run any export command with both flags set, for example `stellar-etl export_transactions --testnet --futurenet --end-ledger 1000` or `stellar-etl get_ledger_range_from_times --testnet --futurenet ...`. The command will export testnet results even though the invocation also requested futurenet.

## Target Code

- `internal/utils/main.go:MustCommonFlags:460-537` — accepts both booleans without validation
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — prioritizes `IsTest` over `IsFuture`
- `cmd/export_transactions.go:Run:19-25` — consumes the resolved environment directly for transaction export
- `cmd/get_ledger_range_from_times.go:Run:42-63` — reproduces the same ambiguous network selection in range lookup

## Evidence

The utility layer defines both flags as independent booleans and does not reject the combination anywhere in the repo. `GetEnvironmentDetails` hard-selects the testnet passphrase, archive URLs, and core config first, and every export command uses that environment immediately to build its backend.

## Anti-Evidence

Operators normally set only one network flag, and the help text describes each as “instead of Mainnet,” which hints they are intended to be exclusive. But the code does not enforce that intent, so a conflicting invocation still produces an apparently successful export from the wrong chain.
