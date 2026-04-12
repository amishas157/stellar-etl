# H001: Viper-loaded common flags are silently ignored by exports

**Date**: 2026-04-12
**Subsystem**: utilities
**Severity**: High
**Impact**: wrong-network / wrong-source export
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When an operator sets common export settings such as `testnet`, `futurenet`, `datastore-path`, `buffer-size`, `num-workers`, `retry-limit`, `retry-wait`, or `write-parquet` in `~/.stellar-etl.yaml` or matching environment variables, the export commands should use those values exactly as if the corresponding CLI flags had been passed. A run configured for testnet should read testnet history/datastore objects, and a run configured with a custom datastore path should export from that configured source instead of pubnet defaults.

## Mechanism

`cmd/root.go:initConfig()` loads config files and environment variables through Viper, but `utils.MustCommonFlags()` never reads from Viper or bound config state; it only calls `flags.Get*()` on Cobra flags. As a result, unset CLI flags fall back to default values (`testnet=false`, `futurenet=false`, default datastore path, default parquet toggle), and downstream helpers like `GetEnvironmentDetails()` and `CreateDatastore()` build a plausible export from the wrong network/source without any warning that the loaded config was ignored.

## Trigger

1. Put `testnet: true` and/or a non-default `datastore-path` in `~/.stellar-etl.yaml` or matching environment variables.
2. Run an archive-backed export such as `stellar-etl export_ledgers --start-ledger <n> --end-ledger <n>` or `stellar-etl export_transactions --start-ledger <n> --end-ledger <n>` **without** passing the corresponding common flags on the CLI.
3. The command acknowledges the config file in `initConfig()`, but the export still uses pubnet/default-datastore behavior because `MustCommonFlags()` returned default flag values.

## Target Code

- `cmd/root.go:initConfig:51-74` — loads config and env through Viper
- `internal/utils/main.go:MustCommonFlags:460-537` — reads only Cobra flag values, never Viper-backed config/env
- `internal/utils/main.go:GetEnvironmentDetails:886-915` — selects pubnet/testnet/futurenet entirely from `CommonFlagValues`
- `internal/utils/main.go:CreateDatastore:990-1006` — derives the datastore bucket path from `CommonFlagValues.DatastorePath`
- `cmd/export_ledgers.go:17-32` — uses `MustCommonFlags()` result to build the ledger input environment

## Evidence

`initConfig()` calls `viper.AutomaticEnv()` and `viper.ReadInConfig()`, but there is no `BindPFlag`, `BindEnv`, or `viper.Get*` path anywhere in the flag-parsing helpers. Every archive export reads `commonArgs := utils.MustCommonFlags(cmd.Flags(), cmdLogger)` and then passes that struct into `GetEnvironmentDetails()` / input readers, so config-only network and datastore selections are silently discarded before any data is read.

## Anti-Evidence

Explicit CLI flags still work correctly, so this only triggers in automation that relies on config files or environment variables instead of command-line arguments. The failure mode is not a crash: it produces a structurally valid export, just from the wrong network/source settings.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of cli-commands/001-config-file-settings-ignored.md
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/cli-commands/001-config-file-settings-ignored.md`
**Failed At**: reviewer

### Trace Summary

This hypothesis is substantively identical to `cli-commands/001-config-file-settings-ignored.md`, which was already investigated and found NOT_VIABLE. Both hypotheses identify the same root cause (Viper loads config but no `BindPFlag`/`Get*` calls bridge it to flag readers), target the same code paths (`root.go:initConfig`, `MustCommonFlags`, `MustFlags`), and claim the same impact (wrong-network exports). The prior investigation conclusively established that the Viper integration is dead `cobra init` scaffolding, not an intentionally designed feature, making this a UX/documentation concern rather than a data correctness bug.

### Code Paths Examined

- `cmd/root.go:initConfig:52-74` — Viper setup loads config and prints banner, values never consumed (confirmed by prior investigation)
- `internal/utils/main.go:MustCommonFlags:460-537` — reads via `flags.Get*()` only, no Viper calls (confirmed by prior investigation)
- Repository-wide grep for `BindPFlag|BindPFlags|BindEnv|viper\.Get` — zero hits outside hypothesis/fail files (confirmed)

### Why It Failed

Exact duplicate of `cli-commands/001-config-file-settings-ignored.md`. That investigation traced the complete Viper/Cobra integration path and found it is incomplete `cobra init` scaffolding: no config schema was designed, documented, or integrated. The README documents all settings exclusively as CLI flags. The `--config` flag and "Using config file:" message are misleading UX artifacts, but the data export pipeline produces correct output through its documented CLI flag interface.

### Lesson Learned

Cross-subsystem duplicate checking is essential. The same Viper dead-scaffolding observation can be filed from the `utilities` angle (focusing on `MustCommonFlags`) or the `cli-commands` angle (focusing on `initConfig` and command wiring), but the root cause and conclusion are identical.
