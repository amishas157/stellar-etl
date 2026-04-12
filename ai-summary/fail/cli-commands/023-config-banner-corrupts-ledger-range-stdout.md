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

Traced the Cobra initialization path: `cobra.OnInitialize(initConfig)` (root.go:38) registers `initConfig` to run before any command's `Run` function. When a config file exists and `viper.ReadInConfig()` succeeds, `initConfig` prints `"Using config file: ..."` to stdout via `fmt.Println` (root.go:73). Then `get_ledger_range_from_times`'s `Run` function executes, and when `path == ""` (line 74), `fmt.Println(string(marshalled))` writes JSON to stdout (line 80). Both writes target `os.Stdout`, producing a two-line output where the first line is non-JSON banner text.

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

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/data_integrity_poc_test.go
**Test Name**: "TestConfigBannerCorruptsLedgerRangeStdout"
**Test Language**: Go

### Demonstration

The test creates a temporary Viper config file, sets the package-level `cfgFile` variable to point at it, then captures `os.Stdout` during execution of `initConfig()` followed by the JSON output path from `get_ledger_range_from_times`. It proves that stdout contains two lines — a non-JSON banner followed by valid JSON — and that `json.Unmarshal` on the full stdout fails with `invalid character 'U' looking for beginning of value`, confirming that any downstream JSON parser would reject this output.

### Test Body

```go
package cmd

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/spf13/viper"
)

// TestConfigBannerCorruptsLedgerRangeStdout demonstrates that initConfig()
// prints a "Using config file: ..." banner to stdout via fmt.Println, which
// corrupts the machine-readable JSON output of get_ledger_range_from_times
// when it writes to stdout (i.e., when -o "" is used).
func TestConfigBannerCorruptsLedgerRangeStdout(t *testing.T) {
	// 1. Create a temporary config file that Viper can read
	tmpDir := t.TempDir()
	cfgPath := filepath.Join(tmpDir, "test-config.yaml")
	err := os.WriteFile(cfgPath, []byte("some_key: some_value\n"), 0644)
	if err != nil {
		t.Fatal(err)
	}

	// Save original state
	oldCfgFile := cfgFile
	oldStdout := os.Stdout
	defer func() {
		cfgFile = oldCfgFile
		os.Stdout = oldStdout
		viper.Reset()
	}()

	// 2. Capture stdout via a pipe
	r, w, err := os.Pipe()
	if err != nil {
		t.Fatal(err)
	}
	os.Stdout = w

	// 3. Point cfgFile at our temp config and run initConfig
	cfgFile = cfgPath
	viper.Reset()
	initConfig()

	// 4. Simulate the command's stdout JSON output path (get_ledger_range_from_times.go:79-80)
	result := ledgerRange{Start: 22343680, End: 22343743}
	marshalled, _ := json.Marshal(result)
	fmt.Println(string(marshalled))

	// 5. Close writer and read captured output
	w.Close()
	var buf bytes.Buffer
	io.Copy(&buf, r)
	capturedOutput := buf.String()

	// Restore stdout so t.Log works
	os.Stdout = oldStdout

	// 6. Assert: stdout is corrupted — it's not valid JSON
	lines := strings.Split(strings.TrimSpace(capturedOutput), "\n")

	// There should be at least two lines: banner + JSON
	if len(lines) < 2 {
		t.Fatalf("Expected multiple lines in stdout (banner + JSON), got %d line(s):\n%s", len(lines), capturedOutput)
	}

	// The first line should be the config banner, not JSON
	if !strings.HasPrefix(lines[0], "Using config file:") {
		t.Fatalf("Expected first line to be config banner, got: %q", lines[0])
	}

	// The full stdout output should fail JSON parsing
	var parsed interface{}
	parseErr := json.Unmarshal([]byte(capturedOutput), &parsed)
	if parseErr == nil {
		t.Fatal("Expected full stdout to be invalid JSON due to config banner, but json.Unmarshal succeeded")
	}

	// Verify the JSON line itself is valid (the data is correct, just the stream is corrupted)
	var rangeResult ledgerRange
	err = json.Unmarshal([]byte(lines[1]), &rangeResult)
	if err != nil {
		t.Fatalf("Expected second line to be valid JSON, but got error: %v", err)
	}
	if rangeResult.Start != 22343680 || rangeResult.End != 22343743 {
		t.Fatalf("JSON data itself is wrong: got %+v", rangeResult)
	}

	t.Logf("DEMONSTRATED: stdout output is corrupted by config banner")
	t.Logf("Full stdout:\n%s", capturedOutput)
	t.Logf("Line 1 (banner): %s", lines[0])
	t.Logf("Line 2 (JSON):   %s", lines[1])
	t.Logf("json.Unmarshal(full_stdout) error: %v", parseErr)
}
```

### Test Output

```
=== RUN   TestConfigBannerCorruptsLedgerRangeStdout
    data_integrity_poc_test.go:94: DEMONSTRATED: stdout output is corrupted by config banner
    data_integrity_poc_test.go:95: Full stdout:
        Using config file: /var/folders/.../test-config.yaml
        {"start":22343680,"end":22343743}
    data_integrity_poc_test.go:96: Line 1 (banner): Using config file: /var/folders/.../test-config.yaml
    data_integrity_poc_test.go:97: Line 2 (JSON):   {"start":22343680,"end":22343743}
    data_integrity_poc_test.go:98: json.Unmarshal(full_stdout) error: invalid character 'U' looking for beginning of value
--- PASS: TestConfigBannerCorruptsLedgerRangeStdout (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.725s
```

---

## Final Review

**Verdict**: REJECTED
**Date**: 2026-04-12
**Final review by**: gpt-5.4, high
**Failed At**: final-review

### Adversarial Analysis

1. **Does the PoC actually exercise the claimed issue?** Yes. I replaced the simulated PoC with a stricter reproduction that builds the real `stellar-etl` binary, runs `stellar-etl --config <tempfile> get_ledger_range_from_times -s 2019-02-06T09:14:43+00:00 -e 2019-02-06T09:20:23+00:00 -o ""`, and captures stdout. The command prints a config banner first and then a valid JSON ledger range, so the mixed stdout behavior is real.
2. **Are the preconditions realistic?** Partially. The trigger requires two non-default choices: a readable config file and an explicit empty `-o ""` to force the undocumented stdout branch. That is plausible for shell automation, but narrower than ordinary use because the documented default path writes to `exported_range.txt`.
3. **Is the behavior a bug or by design?** This is a real CLI output bug, likely inherited from Cobra/Viper scaffolding, not an intended machine-readable interface. But the fact that it is a bug does not make it an in-scope data-integrity finding.
4. **Does the impact match the claimed severity?** No. Final severity is **Informational**, not Medium. The ledger-range JSON values themselves remain correct; the extra banner text makes stdout obviously invalid JSON and causes consumers to fail immediately. This is not silent corruption of plausible data.
5. **Is the finding in scope?** No. The stated objective is silent data corruption that downstream systems can consume without error. Here, downstream JSON parsers fail closed on the first byte of the banner. That is a CLI UX/integration failure, not a data-integrity issue under this review objective.
6. **Is the test itself correct?** Yes. The final reproduction uses the built production binary and the actual Cobra command path, not mocked internals, and it verifies both that the complete stdout stream is invalid JSON and that the JSON line itself is valid.
7. **Can the results be explained without the claimed issue?** No alternative explanation is needed for the mixed stdout behavior itself; it follows directly from two `fmt.Println` calls targeting stdout. The impact, however, is fully explained as a loud parse failure rather than wrong exported data.
8. **Is this finding novel?** Likely novel as phrased, but novelty does not change the scope outcome.

### Rejection Reason

The PoC confirms a real stdout-mixing bug, but it does **not** demonstrate silent or plausible data corruption. The command emits an obviously invalid mixed stream that causes JSON consumers to error out, while the actual ledger-range payload remains correct. That makes this an out-of-scope CLI integration defect for the current data-integrity objective.

### Failed Checks

- 4
- 5
