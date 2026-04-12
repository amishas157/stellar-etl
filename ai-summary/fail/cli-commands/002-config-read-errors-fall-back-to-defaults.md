# H002: Invalid `--config` paths are silently ignored and exports proceed with default settings

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: High
**Impact**: wrong-network / wrong-destination export due to silent config fallback
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

If a user explicitly points `stellar-etl` at a config file with `--config`, the CLI should fail fast when that file is missing, unreadable, or malformed. A command that the operator believes is running with config-provided settings such as `testnet: true`, `output`, or cloud-upload options should not silently continue on pubnet and default output paths.

## Mechanism

`initConfig()` calls `viper.ReadInConfig()` but only handles the success case; any error is discarded. As a result, a typoed `--config` path or malformed YAML produces no error, no warning, and no non-zero exit. The command simply continues with default Cobra flag values, yielding plausible-looking exports from the wrong network or to the wrong output path.

## Trigger

1. Create a config file intended to steer an export, or mistype its path (for example `--config ./tx-testnet.yaml` when the file does not exist).
2. Run an export command while relying on the config for optional settings such as `testnet: true` or a custom `output`.
3. The command completes successfully and emits default-path / default-network output instead of surfacing the config failure.

## Target Code

- `cmd/root.go:initConfig:51-74` — ignores all `viper.ReadInConfig()` failures by checking only `err == nil`
- `internal/utils/main.go:MustCommonFlags:460-537` — falls back to default Cobra flag values after the config read fails
- `internal/utils/main.go:MustArchiveFlags:541-562` — falls back to default output filenames and limits after the config read fails

## Evidence

The only branch after `viper.ReadInConfig()` is:

```go
if err := viper.ReadInConfig(); err == nil {
    fmt.Println("Using config file:", viper.ConfigFileUsed())
}
```

There is no `else`, no logged warning, and no early exit. Because later code reads only Cobra flags, the command naturally keeps running with defaults once the config load fails.

## Anti-Evidence

Because config values are not bound into commands at all, the blast radius partially overlaps with the broader "config file is ignored" issue. But this path is still independently significant: even operators who only want validation of an explicit `--config` file get a false-success run instead of a hard failure.

---

## Review

**Verdict**: NOT_VIABLE
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: FAIL — duplicate of fail/cli-commands/001-config-file-settings-ignored.md
**Failed At**: reviewer

### Trace Summary

Traced `cmd/root.go:initConfig` (lines 52-75): when `cfgFile != ""`, `viper.SetConfigFile(cfgFile)` is called, then `viper.ReadInConfig()` is invoked with the error discarded on failure. The hypothesis correctly identifies that invalid config paths produce no error. However, as established by the prior investigation in fail/001, the entire Viper config integration is dead scaffolding — no `viper.BindPFlag`, `viper.BindPFlags`, or `viper.Get*` calls exist anywhere in the codebase. All 10 export commands read settings exclusively through `pflag.FlagSet.Get*()` in `MustFlags`, `MustCommonFlags`, and `MustArchiveFlags`.

### Code Paths Examined

- `cmd/root.go:initConfig:52-75` — Viper setup loads config file and prints banner; error on line 72 is discarded in `else` branch
- `cmd/root.go:init:37-49` — `--config` persistent flag stores to `cfgFile` variable, never bound to Viper
- `internal/utils/main.go:MustFlags:320-441` — reads all settings via `flags.Get*()` (pflag), zero Viper calls
- `internal/utils/main.go:MustCommonFlags:460-537` — same pflag-only read path
- `internal/utils/main.go:MustArchiveFlags:541-562` — same pflag-only read path

### Why It Failed

This is a duplicate of fail/cli-commands/001. H001 already conclusively established that the Viper config integration is dead Cobra scaffolding code — config file values never reach any command regardless of whether the file loads successfully or fails. H002's specific concern (error handling on invalid config paths) is a strict subset of H001's finding: if even a *successfully loaded* config file has zero effect on execution, then the error-handling behavior for a *failed* config load is irrelevant to data correctness.

The hypothesis frames this as "silent fallback to defaults," but there is no fallback — defaults are the *only* path. The exports produce identical output whether `--config` points to a valid file, an invalid file, or is omitted entirely. No data corruption vector exists because config values are never consumed.

### Lesson Learned

When a prior investigation has established that an entire integration layer (Viper config) is dead code, sub-hypotheses about specific failure modes within that dead code path (error handling, malformed input, missing files) are automatically moot. The decisive question is whether config values reach the execution path at all, not how gracefully config loading fails.
