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
