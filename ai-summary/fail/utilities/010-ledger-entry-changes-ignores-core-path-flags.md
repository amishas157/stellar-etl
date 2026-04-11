# H002: `export_ledger_entry_changes` ignores explicit core path flags

**Date**: 2026-04-11
**Subsystem**: utilities
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `export_ledger_entry_changes` accepts `--core-executable` and `--core-config`, the resulting export should come from that exact Stellar Core binary and config. A run pointed at a custom core config should not silently read from the repo's default docker config instead.

## Mechanism

The command parses `--core-executable` / `--core-config` via `MustCoreFlags()`, but it discards `execPath` immediately and only uses `configPath` for an emptiness check in continuous mode. The actual backend creation then calls `CreateLedgerBackend(ctx, commonArgs.UseCaptiveCore, env)`, where `env` still contains the hardcoded `BinaryPath` and `CoreConfig` from `GetEnvironmentDetails()`. If the hardcoded defaults exist, the export can succeed against the wrong core setup; if they do not, the command fails even though the operator provided valid explicit paths.

## Trigger

Run `stellar-etl export_ledger_entry_changes --captive-core --core-executable /custom/stellar-core --core-config /custom/core.cfg ...` where the supplied paths differ from `/usr/bin/stellar-core` and `/etl/docker/stellar-core*.cfg`.

## Target Code

- `internal/utils/main.go:AddCoreFlags:267-277` — exposes `--core-executable` and `--core-config`
- `internal/utils/main.go:MustCoreFlags:596-627` — parses those flags into `execPath` / `configPath`
- `cmd/export_ledger_entry_changes.go:Run:33-39` — reads the parsed core flags
- `cmd/export_ledger_entry_changes.go:Run:57-63` — only checks `configPath` for emptiness, then creates the backend from `env`
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — hardcodes `BinaryPath` / `CoreConfig`
- `internal/utils/main.go:CreateLedgerBackend:1011-1019` — uses only the hardcoded environment values in captive-core mode

## Evidence

`export_ledger_entry_changes` binds the first `MustCoreFlags()` return value to `_`, so the user-supplied executable path is dropped immediately. The command then ignores the parsed config path during backend creation and instead forwards the unchanged `env` object into `CreateLedgerBackend()`, which only consults `env.BinaryPath` and `env.CoreConfig`.

## Anti-Evidence

If the operator passes the same binary/config paths that `GetEnvironmentDetails()` would have hardcoded anyway, the export will look correct and the bug stays hidden. The command also checks for an empty config in one continuous-export branch, which can make it look like the parsed value is being used more broadly than it really is.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — substantially equivalent to fail/utilities/009-archive-captive-core-hardcodes-docker-paths.md
**Failed At**: reviewer

### Trace Summary

Traced from `export_ledger_entry_changes` init (line 382, registers `AddCoreFlags`), through the `Run` function (line 37, discards `execPath` via `_` and captures `configPath`), to `CreateLedgerBackend(ctx, commonArgs.UseCaptiveCore, env)` (line 62) which uses `env` from `GetEnvironmentDetails(commonArgs)` (lines 886-914). Confirmed that `env.BinaryPath` and `env.CoreConfig` are hardcoded docker paths, and neither the user-supplied `execPath` nor `configPath` reaches `CreateCaptiveCoreBackend()`. However, the critical network parameters (`NetworkPassphrase`, `HistoryArchiveURLs`) are controlled by code, not the config file, so the data produced is always correct for the selected network.

### Code Paths Examined

- `cmd/export_ledger_entry_changes.go:init:379-387` — registers `AddCoreFlags()`, confirming flags are exposed
- `cmd/export_ledger_entry_changes.go:Run:37` — `_, configPath, ... := utils.MustCoreFlags(...)` — `execPath` explicitly discarded
- `cmd/export_ledger_entry_changes.go:Run:57` — `configPath` used only for emptiness check in continuous mode
- `cmd/export_ledger_entry_changes.go:Run:62` — `CreateLedgerBackend(ctx, commonArgs.UseCaptiveCore, env)` — `env` has hardcoded paths
- `internal/utils/main.go:GetEnvironmentDetails:886-914` — hardcodes `BinaryPath="/usr/bin/stellar-core"` and network-specific `CoreConfig`
- `internal/utils/main.go:CreateCaptiveCoreBackend:921-943` — uses `e.CoreConfig` for TOML loading and `e.BinaryPath` for binary path; critically, `NetworkPassphrase` and `HistoryArchiveURLs` are passed separately via `CaptiveCoreTomlParams` (lines 924-926) and `CaptiveCoreConfig` (lines 937-938), overriding anything in the TOML file

### Why It Failed

1. **Not a data correctness bug.** The hypothesis correctly identifies that `execPath` is discarded and `configPath` isn't forwarded to the backend. However, ledger data is determined by Stellar network consensus, not by which stellar-core binary or config file is used. `CreateCaptiveCoreBackend()` explicitly overrides network-critical parameters (`NetworkPassphrase`, `HistoryArchiveURLs`) from the code-controlled `EnvironmentDetails` struct, so the TOML config file only affects operational aspects (database paths, peer connections, etc.), not the data content of exported ledgers.

2. **Substantially equivalent to 009.** The prior investigation (fail/utilities/009) analyzed the identical core code paths (`GetEnvironmentDetails` → `CreateLedgerBackend` → `CreateCaptiveCoreBackend`) and reached the same conclusion: hardcoded captive-core paths are an operational/usability issue in a deprecated feature, not a source of silent data corruption. H002 examines the same root cause from a different entry point (`export_ledger_entry_changes` vs archive commands) but the data correctness analysis is identical.

3. **Deprecated feature.** The `--captive-core` flag is explicitly deprecated ("Will be removed in the Protocol 23 update"). The primary backend is the DataStore (GCS bucket) path. The ignored flags are part of a deprecated subsystem.

### Lesson Learned

When a hypothesis about ignored CLI flags touches the same underlying code paths as a prior investigation (here, `GetEnvironmentDetails` → `CreateLedgerBackend`), check whether the prior analysis already covers the data correctness implications. Different entry points to the same root cause yield the same verdict.
