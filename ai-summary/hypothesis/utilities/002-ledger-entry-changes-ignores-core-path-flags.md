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
