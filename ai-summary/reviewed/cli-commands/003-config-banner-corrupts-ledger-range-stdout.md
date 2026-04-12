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

---

## Review

**Verdict**: VIABLE
**Severity**: Informational
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the Cobra initialization path: `cobra.OnInitialize(initConfig)` (root.go:38) registers `initConfig` to run before any command's `Run` function. When a config file exists and `viper.ReadInConfig()` succeeds, `initConfig` prints `"Using config file: ..."` to stdout via `fmt.Println` (root.go:73). Then `get_ledger_range_from_times`'s `Run` function executes, and when `path == ""` (line 74), it prints the JSON result to stdout via `fmt.Println` (line 80). Both writes target `os.Stdout`, producing a two-line output where the first line is non-JSON banner text.

### Code Paths Examined

- `cmd/root.go:init:38` — `cobra.OnInitialize(initConfig)` ensures `initConfig` runs before any command
- `cmd/root.go:initConfig:52-74` — loads config file; on success, `fmt.Println("Using config file:", viper.ConfigFileUsed())` writes to stdout
- `cmd/get_ledger_range_from_times.go:74-80` — when `path == ""`, `fmt.Println(string(marshalled))` writes JSON to stdout
- `cmd/get_ledger_range_from_times.go:90` — default output flag is `"exported_range.txt"`, so stdout mode requires explicit `-o ""`

### Findings

The bug is real — both `initConfig` and the command's stdout branch write to `os.Stdout` via `fmt.Println`, and `initConfig` runs first via `cobra.OnInitialize`. When a config file exists and the user passes `-o ""`, stdout contains:

```
Using config file: /path/to/.stellar-etl.yaml
{"start":123,"end":456}
```

A JSON parser reading this would fail immediately on the first line.

**Severity downgrade from Medium to Informational**: The hypothesis correctly identifies a real code defect, but the impact does not meet the "silent data corruption" threshold for the data correctness objective:

1. **Not silent**: The prepended banner makes the output obviously invalid JSON. Any JSON parser (`jq`, `json.loads`, `json.Unmarshal`) will reject it with a parse error. This is a detectable failure, not a case where wrong-but-plausible data is silently consumed.
2. **Data content is correct**: The JSON object itself (`{"start":N,"end":M}`) contains the correct ledger range values. No financial or structural data is wrong.
3. **Narrow trigger**: Requires both (a) a config file at `~/.stellar-etl.yaml` or via `--config`, AND (b) explicit `-o ""` to activate the stdout branch. The default output path writes to `exported_range.txt`, unaffected by the banner.
4. **UX/integration bug**: This is scaffolding-originated — the Viper config banner is from Cobra's `init` template and should print to stderr (or not at all). It's the same dead scaffolding identified in fail/cli-commands/001, but H003 identifies a different consequence: stdout corruption rather than config value inertness.

### PoC Guidance

- **Test file**: `cmd/get_ledger_range_from_times_test.go` (or create a new test file if none exists)
- **Setup**: Create a temporary config file. Set `cfgFile` package variable (or use `--config` flag) to point to it. Set up the command with `-o ""` and valid start/end times that can be mocked.
- **Steps**: Capture stdout during command execution. Parse stdout as JSON.
- **Assertion**: Assert that `json.Unmarshal(stdout_bytes, &result)` fails when a config file is present, demonstrating the banner corruption. Alternatively, assert that stdout contains more than one line or that the first line is not valid JSON.
