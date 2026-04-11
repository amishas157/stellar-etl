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
