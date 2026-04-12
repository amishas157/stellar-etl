# H002: Viper-loaded archive range/output flags are silently ignored

**Date**: 2026-04-12
**Subsystem**: utilities
**Severity**: High
**Impact**: wrong ledger-range export
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If `start-ledger`, `limit`, `output`, or `parquet-output` are provided through the config file or environment, archive-style commands should honor those settings and export the requested slice of ledgers into the configured artifact paths. A config-only `start-ledger` / `limit` combination should not silently expand into "start at ledger 2 and export everything."

## Mechanism

`MustArchiveFlags()` reads `start-ledger`, `output`, `parquet-output`, and `limit` directly from Cobra flags via `flags.Get*()` and never consults the Viper state loaded in `initConfig()`. Commands like `export_transactions` and `export_ledgers` then pass those defaults into the input readers, so config/env-provided ledger bounds and row limits are dropped and the command emits a plausible but wrong dataset into default filenames.

## Trigger

1. Put values such as `start-ledger: 60000000`, `limit: 1`, `output: custom-transactions.json`, and `parquet-output: custom-transactions.parquet` in `~/.stellar-etl.yaml` or matching environment variables.
2. Run `stellar-etl export_transactions --end-ledger 60000010` without passing the archive flags on the CLI.
3. The command still uses `start-ledger=2`, `limit=-1`, and the default output filenames returned by `MustArchiveFlags()`, exporting the wrong ledger window and artifact paths without surfacing any configuration error.

## Target Code

- `cmd/root.go:initConfig:51-74` — loads config and env through Viper
- `internal/utils/main.go:MustArchiveFlags:540-562` — resolves range/path settings only from Cobra flags
- `cmd/export_transactions.go:17-25` — consumes `MustArchiveFlags()` and sends those values to `input.GetTransactions()`
- `cmd/export_ledgers.go:17-23` — same helper path for ledger exports

## Evidence

`MustArchiveFlags()` has no Viper integration at all: it always calls `flags.GetUint32("start-ledger")`, `flags.GetString("output")`, `flags.GetString("parquet-output")`, and `flags.GetInt64("limit")`. The archive export commands immediately trust those returned values when constructing the reader request, so config/env-only range selection cannot affect the exported rows.

## Anti-Evidence

Commands still behave correctly when the operator passes archive flags explicitly on the CLI. The bug is specifically the silent mismatch between "config file loaded" and "archive helper still using defaults."

---

## Review

**Verdict**: NOT_VIABLE — duplicate of cli-commands/001-config-file-settings-ignored.md and utilities/001-viper-common-flags-ignored.md
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of `ai-summary/fail/cli-commands/001-config-file-settings-ignored.md`
**Failed At**: reviewer

### Trace Summary

This hypothesis targets `MustArchiveFlags()` (lines 540-562) for the same Viper dead-scaffolding root cause already investigated twice: once as `cli-commands/001-config-file-settings-ignored.md` (which explicitly listed `MustArchiveFlags:541-562` in its code paths examined) and once as `utilities/001-viper-common-flags-ignored.md` (which was itself marked duplicate of the cli-commands finding). The prior investigation confirmed that the entire Viper integration is unused `cobra init` boilerplate — no `BindPFlag`, `BindPFlags`, or `viper.Get*` calls exist anywhere in the codebase. `MustArchiveFlags` is just one of three flag-reader functions (alongside `MustFlags` and `MustCommonFlags`) that all share the same pflag-only design.

### Code Paths Examined

- `internal/utils/main.go:MustArchiveFlags:540-562` — reads `start-ledger`, `output`, `parquet-output`, `limit` via `flags.Get*()` only; no Viper calls (confirmed, already documented in cli-commands/001)
- `cmd/root.go:initConfig:52-74` — Viper setup loads config and prints banner; values never consumed (already documented in cli-commands/001 and utilities/001)

### Why It Failed

Exact duplicate. The prior investigation `cli-commands/001-config-file-settings-ignored.md` already traced all three flag-reader functions (`MustFlags`, `MustCommonFlags`, `MustArchiveFlags`) and concluded that the Viper integration is incomplete Cobra scaffolding, not a designed feature. The config file interface is undocumented, unsupported, and no config schema exists. The data export pipeline produces correct output through its documented CLI flag interface. H002 simply narrows the scope from "all flags" to "archive flags" without any new insight.

### Lesson Learned

When a prior investigation covers the complete flag-reading surface (all three `Must*Flags` functions), a narrower hypothesis targeting one specific function within that surface adds no new information and should be detected as a duplicate during novelty checking.
