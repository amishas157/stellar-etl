# H001: Archive exports hardcode captive-core docker paths

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If an archive-style export command accepts `--captive-core`, the command should either expose `--core-executable` / `--core-config` and honor those values, or reject captive-core mode when those paths are unavailable. Running `export_transactions`, `export_operations`, `export_effects`, `export_trades`, `export_contract_events`, or `export_token_transfer` with `--captive-core` should therefore use a captive-core binary and config that actually match the operator's environment.

## Mechanism

`AddCommonFlags()` registers `--captive-core` for every command, but the archive export commands only add common/archive/cloud flags and never add the core-path flags. `GetEnvironmentDetails()` then injects fixed docker paths (`/usr/bin/stellar-core` and `/etl/docker/stellar-core*.cfg`), and `CreateLedgerBackend()` / `CreateCaptiveCoreBackend()` use those fixed values verbatim. As a result, archive exports in captive-core mode either fail outside the container image or silently run against the hardwired docker config instead of the operator's intended setup.

## Trigger

Run any archive export such as `stellar-etl export_transactions --captive-core -e 12345` on a host where the Stellar Core binary/config do not live at the hardcoded docker paths, or where the desired captive-core config differs from `/etl/docker/stellar-core*.cfg`.

## Target Code

- `internal/utils/main.go:AddCommonFlags:232-245` — exposes `--captive-core` on every command
- `cmd/export_transactions.go:init:70-75` — archive export registers common/archive/cloud flags but no `AddCoreFlags`
- `cmd/export_token_transfers.go:init:66-71` — same missing core-path wiring on another archive export
- `cmd/export_contract_events.go:init:71-78` — same missing core-path wiring on contract-event export
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — hardcodes captive-core binary/config paths per network
- `internal/utils/main.go:CreateCaptiveCoreBackend:921-943` — launches captive core using those hardcoded paths
- `internal/utils/main.go:CreateLedgerBackend:1011-1019` — captive-core mode always routes through the hardcoded environment details

## Evidence

The CLI surface advertises `--captive-core` through `AddCommonFlags()`, but the archive exporters shown above never register `AddCoreFlags()`, so there is no command-line path for the operator to override the binary/config that captive-core will use. The only values that can reach `CreateCaptiveCoreBackend()` come from `GetEnvironmentDetails()`, which always fills in docker-specific defaults.

## Anti-Evidence

The bug is masked inside environments that happen to mount Stellar Core at `/usr/bin/stellar-core` and use the repo's docker config files. The flag is also marked deprecated, so some operators may never exercise this path.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated
**Failed At**: reviewer

### Trace Summary

Traced the full execution path from `AddCommonFlags()` (line 238, registers `--captive-core`) through `MustFlags()`/`MustCommonFlags()` (reads the flag into `UseCaptiveCore`), into `GetEnvironmentDetails()` (lines 886-914, hardcodes `/usr/bin/stellar-core` and `/etl/docker/stellar-core*.cfg`), and finally into `CreateLedgerBackend()` → `CreateCaptiveCoreBackend()` (lines 1011-1019, 921-943). Confirmed that archive export commands never register `AddCoreFlags()` and that the hardcoded docker paths are the only values that reach the captive core backend. However, the failure mode is loud, not silent.

### Code Paths Examined

- `internal/utils/main.go:AddCommonFlags:232-245` — registers `--captive-core` with deprecated warning
- `internal/utils/main.go:MustFlags:320-430` — reads `captive-core` flag into `UseCaptiveCore` bool
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — hardcodes `BinaryPath="/usr/bin/stellar-core"` and network-specific `CoreConfig` for all three networks
- `internal/utils/main.go:CreateCaptiveCoreBackend:921-943` — uses `e.CoreConfig` for TOML loading (line 923) and `e.BinaryPath` for the backend config (line 935); errors from `NewCaptiveCoreTomlFromFile` and `NewCaptive` are returned to caller
- `internal/utils/main.go:CreateLedgerBackend:1011-1019` — when `useCaptiveCore` is true, calls `env.CreateCaptiveCoreBackend()` and returns errors
- `internal/input/transactions.go:GetTransactions:23-29` — passes error from `CreateLedgerBackend` back to caller
- `cmd/export_transactions.go:Run:25-28` — calls `cmdLogger.Fatal()` on any error from `GetTransactions`

### Why It Failed

This is **not a data correctness bug**. The hypothesis claims "silent failure" but the actual failure mode is loud:

1. **Outside docker**: If `/usr/bin/stellar-core` or `/etl/docker/stellar-core*.cfg` don't exist, `NewCaptiveCoreTomlFromFile()` or `NewCaptive()` return an error. This error propagates through `CreateCaptiveCoreBackend()` → `CreateLedgerBackend()` → `GetTransactions()` → `cmdLogger.Fatal()`. The command crashes with a clear error message. No data is produced.

2. **Inside docker**: The hardcoded paths are the intended paths for the docker deployment. The config files exist in the repository at `docker/stellar-core.cfg`, `docker/stellar-core_testnet.cfg`, and `docker/stellar-core_futurenet.cfg`. Data produced is correct.

3. **No silent corruption scenario**: There is no code path where wrong captive-core paths lead to producing incorrect-but-plausible output. Either the paths resolve to a working stellar-core installation (producing correct data) or they don't (producing a fatal error with zero output).

The `--captive-core` flag is also explicitly deprecated ("Will be removed in the Protocol 23 update"), and the primary backend is now the DataStore (GCS bucket) path. This is an operational/usability limitation in a deprecated feature, not a data correctness issue.

### Lesson Learned

Hardcoded paths that cause loud startup failures are operational annoyances, not data correctness bugs. For the objective of finding silent data corruption, focus on code paths where incorrect values flow through to output rather than code paths that fail before producing any output.
