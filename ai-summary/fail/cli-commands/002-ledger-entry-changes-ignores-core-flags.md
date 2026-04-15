# H001: `export_ledger_entry_changes` ignores `--core-config` and `--core-executable`

**Date**: 2026-04-15
**Subsystem**: cli-commands
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When a caller supplies `--core-config` and `--core-executable`, the ledger-change exporter should build its captive-core backend from those exact paths so the emitted change files reflect the user-selected Core binary, archive configuration, and network state.

## Mechanism

`export_ledger_entry_changes` parses the core flags but discards the executable path and uses `configPath` only for the unbounded-mode nil check. The actual backend comes from `utils.GetEnvironmentDetails(commonArgs)` plus `utils.CreateLedgerBackend(...)`, and that path hard-codes `/usr/bin/stellar-core` and `/etl/docker/stellar-core*.cfg`; if those defaults exist, the command silently exports changes from the default docker config instead of the caller's requested Core config.

## Trigger

Run `export_ledger_entry_changes --captive-core --core-config /path/custom.cfg --core-executable /opt/custom-core ...` where the custom config points at a different archive set or Core state than the built-in docker config. The export will still use the hard-coded default paths.

## Target Code

- `cmd/export_ledger_entry_changes.go:Run:31-63` — parses core flags, but only keeps `configPath` for one validation branch before creating the backend from `env`
- `internal/utils/main.go:MustCoreFlags:596-627` — reads `core-executable` and `core-config`, proving the CLI accepts them
- `internal/utils/main.go:GetEnvironmentDetails:886-915` — overwrites `BinaryPath` and `CoreConfig` with hard-coded network defaults
- `internal/utils/main.go:CreateCaptiveCoreBackend:921-942` — always consumes `e.BinaryPath` and `e.CoreConfig` from the hard-coded environment details

## Evidence

The assignment `_, configPath, startNum, batchSize, outputFolder, parquetOutputFolder := utils.MustCoreFlags(...)` drops the executable path immediately. After that, `env := utils.GetEnvironmentDetails(commonArgs)` populates `BinaryPath` and `CoreConfig` from fixed docker paths, and `utils.CreateLedgerBackend(ctx, commonArgs.UseCaptiveCore, env)` never receives the user-provided core-flag values.

## Anti-Evidence

If the hard-coded docker config or binary is missing, this path fails loudly instead of exporting wrong data. Silent corruption requires the defaults to exist and point at a different Core/archive setup than the one the caller intended to select with the CLI flags.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-15
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full path from flag registration (`AddCoreFlags`) through parsing (`MustCoreFlags`) to backend creation (`GetEnvironmentDetails` → `CreateLedgerBackend` → `CreateCaptiveCoreBackend`). Confirmed that `--core-executable` is discarded and `--core-config` is used only for one unbounded-mode validation check. However, this is consistent with the codebase-wide design: ALL 10 export commands use `GetEnvironmentDetails` with hard-coded paths, no command ever overrides environment details from flags, and a TODO (HUBBLE-386) explicitly acknowledges this as a known design limitation to be refactored.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:Run:31-62` — `execPath` assigned to `_`, `configPath` used only at line 57 validation
- `internal/utils/main.go:GetEnvironmentDetails:886-915` — hard-codes `BinaryPath` and `CoreConfig` for all three networks
- `internal/utils/main.go:CreateLedgerBackend:1011-1041` — passes `env` (with hard-coded paths) to `CreateCaptiveCoreBackend`
- `internal/utils/main.go:AddCoreFlags:267-277` — registers the flags but they are vestigial
- `cmd/export_contract_events.go:21` — TODO comment: "GetEnvironmentDetails should be refactored" (HUBBLE-386)
- All other export commands (`export_ledgers.go`, `export_transactions.go`, etc.) — none register core flags, all use `GetEnvironmentDetails` identically

### Why It Failed

This is **working-as-designed behavior**, not structural data corruption. The codebase was never designed to use user-provided core paths for backend creation — `GetEnvironmentDetails` is the canonical and sole source of environment configuration across all 10 export commands. The `--core-config` and `--core-executable` flags on `export_ledger_entry_changes` are vestigial/planned features that were registered but never wired into the backend creation path. The exported data is correct for the selected network (pubnet/testnet/futurenet as chosen by `--testnet`/`--futurenet` flags). Network passphrase, archive URLs, and core config are all consistently set per-network. The flags being dead does not produce wrong data — it produces the correct data for the default network config while ignoring customization that was never implemented.

### Lesson Learned

When flags appear to be ignored, check whether ANY command in the codebase actually uses those flags for the claimed purpose. If no command does, the flags are likely planned-but-unimplemented features, not a data correctness bug. A known design limitation with a tracking ticket (HUBBLE-386) is not a data corruption finding.
