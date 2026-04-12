# H004: Change export ignores core binary and config overrides

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`export_ledger_entry_changes --captive-core --core-executable X --core-config Y` should either run captive core with exactly those paths or fail if they are unusable. The exported account / trustline / contract-data / config-setting rows should therefore come from the operator-selected core build and TOML, not from hard-coded defaults.

## Mechanism

`MustCoreFlags()` parses both `--core-executable` and `--core-config`, but `export_ledger_entry_changes` discards `execPath` entirely and uses `configPath` only for an empty-string check in continuous mode. The actual backend is built via `utils.CreateLedgerBackend(ctx, commonArgs.UseCaptiveCore, env)`, and `env` comes from `GetEnvironmentDetails()`, which hard-codes `/usr/bin/stellar-core` plus `/etl/docker/stellar-core*.cfg`, so a user can request one core/config pair while the export silently runs against another.

## Trigger

1. Invoke `stellar-etl export_ledger_entry_changes --captive-core --core-executable /tmp/custom-core --core-config /tmp/custom.toml --start-ledger L --end-ledger M`.
2. Provide a custom TOML or binary that should change the prepared network/archive behavior or should fail if the file path is invalid.
3. Observe that the command still builds captive core from the default environment paths instead of the supplied overrides.

## Target Code

- `cmd/export_ledger_entry_changes.go:31-69` — parses `MustCoreFlags()`, throws away `execPath`, and creates the backend from `env`
- `internal/utils/main.go:GetEnvironmentDetails:886-915` — hard-codes `BinaryPath` and `CoreConfig` by network selection
- `internal/utils/main.go:CreateLedgerBackend:1009-1040` — uses `env.CreateCaptiveCoreBackend()` rather than the values parsed from CLI core flags
- `internal/input/changes.go:32-79` — shows the separate `PrepareCaptiveCore(execPath, tomlPath, ...)` helper that would have honored these user-supplied paths, but this command never calls it

## Evidence

The command destructures `MustCoreFlags()` as `_, configPath, ...`, proving `execPath` is intentionally ignored in the current code. The only place `configPath` is referenced is `if configPath == "" && commonArgs.EndNum == 0`, while `GetEnvironmentDetails()` always fills `BinaryPath` and `CoreConfig` from baked-in defaults before `CreateLedgerBackend()` constructs captive core.

## Anti-Evidence

If the operator uses the default packaged core binary and TOML, the bug is invisible because the ignored overrides happen to match the hard-coded environment values. The impact appears only when callers intentionally rely on non-default captive-core paths.
