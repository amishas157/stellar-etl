# H003: `export_contract_events` ignores config/env for optional full-bundle flags

**Date**: 2026-04-12
**Subsystem**: utilities
**Severity**: High
**Impact**: wrong row count / wrong artifact configuration
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Once the required CLI bounds are supplied, `export_contract_events` should honor config/env-provided optional settings such as `limit`, `output`, `parquet-output`, `write-parquet`, `extra-fields`, and `cloud-*` exactly the same way as explicit flags. A config that requests `limit: 1`, a custom output path, and parquet generation should produce a one-row contract-event export at those configured destinations.

## Mechanism

`export_contract_events` uses `utils.MustFlags()` for its full flag bundle and `utils.MustCommonFlags()` for the parquet toggle, but both helpers read only from Cobra flags. Because `cmd/root.go:initConfig()` loads Viper state that is never bound into those helpers, the command can print `Using config file: ...` and still fall back to default `limit`, default output filenames, empty upload provider, and `write-parquet=false`, producing a success-shaped export that silently ignores the requested optional behavior.

## Trigger

1. Put values such as `limit: 1`, `output: contract-events.json`, `parquet-output: contract-events.parquet`, `write-parquet: true`, or `cloud-provider: gcp` in `~/.stellar-etl.yaml` or matching environment variables.
2. Run `stellar-etl export_contract_events --start-ledger <n> --end-ledger <n>` with only the required bounds on the CLI.
3. The command still takes default optional values from `MustFlags()` / `MustCommonFlags()`: it exports all matching events, writes to the default filenames, and skips parquet/upload behavior unless those options are also repeated on the command line.

## Target Code

- `cmd/root.go:initConfig:51-74` — loads config and env through Viper
- `internal/utils/main.go:MustFlags:320-440` — reads full command settings only from Cobra flags
- `internal/utils/main.go:MustCommonFlags:460-537` — separately resolves `write-parquet` only from Cobra flags
- `cmd/export_contract_events.go:17-25` — combines `MustFlags()` and `MustCommonFlags()` to drive the command
- `cmd/export_contract_events.go:43-65` — uses the parsed `limit`, output paths, parquet flag, and upload settings to shape the final artifacts

## Evidence

Unlike the archive exporters, `export_contract_events` does not use `MustArchiveFlags()` / `MustCloudStorageFlags()` separately; it relies on `MustFlags()` to gather the whole bundle. That helper never calls Viper, and the command then directly feeds `cmdArgs.StartNum`, `cmdArgs.EndNum`, `cmdArgs.Limit`, `cmdArgs.Path`, `cmdArgs.ParquetPath`, `cmdArgs.Bucket`, `cmdArgs.Credentials`, and `cmdArgs.Provider` into the reader/export/upload path.

## Anti-Evidence

`start-ledger` and `end-ledger` are marked as required CLI flags, so config/env cannot satisfy those two fields by themselves; the trigger requires passing the required bounds on the CLI and relying on config/env only for the optional settings. Explicit CLI values still behave correctly.

---

## Review

**Verdict**: NOT_VIABLE — duplicate of cli-commands/001-config-file-settings-ignored.md and utilities/001-viper-common-flags-ignored.md
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/cli-commands/001-config-file-settings-ignored.md` and `ai-summary/fail/utilities/001-viper-common-flags-ignored.md`
**Failed At**: reviewer

### Trace Summary

This hypothesis targets the exact same root cause as two prior investigations: Viper config/env loading in `root.go:initConfig` is dead Cobra scaffolding that never bridges to flag readers. H001 (utilities) was itself marked as a duplicate of cli-commands/001. H003 merely narrows the focus to `export_contract_events` and `MustFlags()`, but the mechanism, root cause, and conclusion are identical: no `viper.BindPFlag`, `viper.BindPFlags`, or `viper.Get*` calls exist anywhere in the codebase, so all config file values are silently discarded.

### Code Paths Examined

- `cmd/root.go:initConfig:52-74` — Viper setup loads config file and prints banner; values never consumed (confirmed by prior investigations)
- `internal/utils/main.go:MustFlags:320-441` — reads via `flags.Get*()` only, no Viper calls (confirmed by prior investigations)
- `internal/utils/main.go:MustCommonFlags:460-537` — same pflag-only read path (confirmed by prior investigations)

### Why It Failed

Exact duplicate of two prior findings. The cli-commands/001 investigation traced the complete Viper/Cobra integration path and conclusively established that the Viper integration is incomplete `cobra init` scaffolding — no config schema was designed, documented, or integrated. The README documents all settings exclusively as CLI flags. The `--config` flag and "Using config file:" message are misleading UX artifacts, but the data export pipeline produces correct output through its documented CLI flag interface. Narrowing the same observation to `export_contract_events` specifically does not change the analysis or conclusion.

### Lesson Learned

Cross-subsystem and cross-command duplicate checking is essential. The Viper dead-scaffolding observation can be filed from any angle (specific command, specific flag helper, specific subsystem), but the root cause and conclusion are always identical: no code bridges Viper state to Cobra flag readers.
