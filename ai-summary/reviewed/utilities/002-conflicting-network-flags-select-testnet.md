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

Operators normally set only one network flag, and the help text describes each as "instead of Mainnet," which hints they are intended to be exclusive. But the code does not enforce that intent, so a conflicting invocation still produces an apparently successful export from the wrong chain.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

I traced both flag-parsing paths (`MustFlags` at line 320 and `MustCommonFlags` at line 460) and confirmed that `isTest` and `isFuture` are read as independent booleans with no cross-validation. Both parsers store both values into their respective structs (`FlagValues`, `CommonFlagValues`) without checking for mutual exclusion. `GetEnvironmentDetails` (line 886) uses an `if/else if` chain that always evaluates `IsTest` first, so when both are true, testnet configuration is silently returned. The same pattern exists in `get_ledger_range_from_times.go` (line 42-50) which reads both flags and passes them to `GetLedgerRange`, which constructs a `CommonFlagValues` and calls `GetEnvironmentDetails` — same silent prioritization.

### Code Paths Examined

- `internal/utils/main.go:AddCommonFlags:232-246` — registers `--testnet` and `--futurenet` as independent boolean flags with no `MarkMutuallyExclusive` or equivalent
- `internal/utils/main.go:MustFlags:320-440` — reads both flags, stores in `FlagValues`, no validation of `IsTest && IsFuture`
- `internal/utils/main.go:MustCommonFlags:460-537` — reads both flags, stores in `CommonFlagValues`, no validation of `IsTest && IsFuture`
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — `if IsTest` checked first; `else if IsFuture` never reached when both true; returns testnet passphrase, archive URLs, and core config
- `cmd/get_ledger_range_from_times.go:Run:42-50` — reads both booleans independently and passes them to `input.GetLedgerRange`
- `internal/input/ledger_range.go:GetLedgerRange:32-68` — constructs `CommonFlagValues{IsTest, IsFuture}` and calls `GetEnvironmentDetails`, same silent prioritization

### Findings

The bug is confirmed: no code path anywhere in the repository validates that `--testnet` and `--futurenet` are mutually exclusive. When both are set, every export command and `get_ledger_range_from_times` will silently use testnet configuration (passphrase, archive URLs, core config). The exported dataset is structurally valid but from the wrong network.

Severity is downgraded from High to Medium because:
1. The trigger requires explicit operator error (passing both flags), which is unlikely in routine use
2. The output is internally consistent — it is valid testnet data, not corrupted data
3. The practical risk is in scripted/automated invocations where a flag combination bug could go unnoticed

Despite the unlikely trigger, this is a real missing-validation bug where the code silently resolves an ambiguous configuration instead of failing fast.

### PoC Guidance

- **Test file**: `internal/utils/main_test.go` (or create a new test function)
- **Setup**: Construct a `CommonFlagValues` with both `IsTest: true` and `IsFuture: true`
- **Steps**: Call `GetEnvironmentDetails(commonFlags)` and inspect the returned `EnvironmentDetails`
- **Assertion**: Assert that the returned `Network` is `"testnet"` (not `"futurenet"`) AND that the `NetworkPassphrase` matches the testnet passphrase — confirming the silent prioritization. Alternatively, assert that a proper validation function (if added) returns an error when both flags are set.
