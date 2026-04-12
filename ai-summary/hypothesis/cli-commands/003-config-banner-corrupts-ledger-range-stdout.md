# H003: Config-file banner prepends non-JSON text to `get_ledger_range_from_times` stdout output

**Date**: 2026-04-12
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: machine-readable utility output corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

When `get_ledger_range_from_times` is asked to write to stdout instead of a file, stdout should contain exactly one JSON object like `{"start":123,"end":456}`. Scripts that pipe this command into `jq`, shell substitutions, or other automation should not receive extra banner text mixed into the payload.

## Mechanism

Before any command runs, `initConfig()` prints `Using config file: ...` directly to stdout whenever Viper successfully reads a config file. `get_ledger_range_from_times` also emits its result with `fmt.Println(string(marshalled))` when `output == ""`. With both paths active, stdout becomes a two-line mixed stream of banner text plus JSON rather than the single JSON object the utility branch is supposed to return.

## Trigger

1. Ensure a readable config file exists (`~/.stellar-etl.yaml` or `--config cfg.yaml`).
2. Run `stellar-etl --config cfg.yaml get_ledger_range_from_times -s 2019-09-13T23:00:00+00:00 -e 2019-09-14T13:35:10+00:00 -o ""`.
3. Stdout begins with `Using config file: ...` and only then prints the JSON range, so a JSON parser reading stdout sees invalid input.

## Target Code

- `cmd/root.go:initConfig:72-73` — prints the config banner directly to stdout
- `cmd/get_ledger_range_from_times.go:74-80` — sends the ledger-range JSON to stdout when `output` is empty

## Evidence

Both code paths use `fmt.Println(...)` rather than Cobra output streams or the logger. There is no gating that suppresses the config banner when a command intends to return machine-readable stdout.

## Anti-Evidence

The default behavior of `get_ledger_range_from_times` is to write to `exported_range.txt`, so the corruption only appears when callers intentionally request stdout with `-o ""`. That limits exposure, but the stdout branch is explicit production code and is a realistic automation path for shell pipelines.
