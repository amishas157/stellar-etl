# 008: Continuous change export prepares a bounded `[start,0]` range

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness: continuous ledger-change export exits instead of streaming live batches
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes` documents that omitting `--end-ledger` enables continuous export, but the command prepares the backend with `ledgerbackend.BoundedRange(startNum, 0)` before it rewrites `EndNum` for its own loop. That commits the backend to a backwards bounded range and causes the first live ledger read to fail immediately instead of streaming new batches.

The issue is reproducible in the default datastore-backed CLI path. Running `go run . export_ledger_entry_changes -s 2 -c dummy -o /tmp/etl-poc-out` exits with `requested sequence beyond current LedgerRange` on the first ledger read instead of staying alive and exporting changes as new ledgers arrive.

## Root Cause

At `cmd/export_ledger_entry_changes.go:67`, the command always calls:

```go
err = backend.PrepareRange(ctx, ledgerbackend.BoundedRange(startNum, commonArgs.EndNum))
```

When `--end-ledger` is omitted, `commonArgs.EndNum` is still `0`, so the prepared range becomes bounded `[start,0]`. Only afterwards does the command rewrite `commonArgs.EndNum` to `math.MaxInt32`, but that does not change the backend's already prepared range. When `input.StreamChanges()` asks for the first ledger, the buffered backend rejects it because `sequence > to`.

This is inconsistent with the codebase's own intended pattern in `internal/input/changes.go`, where `PrepareCaptiveCore()` starts with `ledgerbackend.UnboundedRange(start)` and only switches to `BoundedRange(start, end)` when `end != 0`.

## Reproduction

During normal operation, the default CLI path reproduces the issue directly:

```bash
cd <repo-root>
go run . export_ledger_entry_changes -s 2 -c dummy -o /tmp/etl-poc-out
```

Observed result during final review:

- the command exited immediately with `unable to create change reader for ledger 2: error getting ledger from the backend: requested sequence beyond current LedgerRange`

## Affected Code

- `cmd/export_ledger_entry_changes.go:61-78` — prepares `BoundedRange(startNum, 0)` before rewriting `EndNum` for pseudo-unbounded looping
- `internal/input/changes.go:111-113` — first live ledger read propagates the backend range failure through `logger.Fatal`
- `internal/input/changes.go:59-75` — contrasting in-repo implementation that correctly uses `UnboundedRange(start)` when `end == 0`
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/range.go:76-83` — `BoundedRange()` always stays bounded; only `UnboundedRange()` enables streaming semantics
- `/Users/amisha.singla/go/pkg/mod/github.com/stellar/go-stellar-sdk@v0.0.0-20251211085638-ba09a6a91775/ingest/ledgerbackend/buffered_storage_backend.go:142-145` — bounded backends reject `GetLedger(sequence)` when `sequence > to`

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestContinuousModePreparesNonsensicalBoundedRange`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package input

import (
	"context"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"
	"time"
)

func TestContinuousModePreparesNonsensicalBoundedRange(t *testing.T) {
	cwd, err := os.Getwd()
	if err != nil {
		t.Fatalf("get working directory: %v", err)
	}

	repoRoot := filepath.Dir(filepath.Dir(cwd))
	outputDir := filepath.Join(t.TempDir(), "changes")

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()

	// The command only requires a non-empty core-config flag in continuous mode.
	cmd := exec.CommandContext(ctx,
		"go", "run", ".", "export_ledger_entry_changes",
		"-s", "2",
		"-c", "dummy",
		"-o", outputDir,
	)
	cmd.Dir = repoRoot

	out, err := cmd.CombinedOutput()
	if ctx.Err() == context.DeadlineExceeded {
		t.Fatalf("command hung instead of failing fast; output so far:\n%s", out)
	}
	if err == nil {
		t.Fatalf("expected continuous export to fail, but it succeeded:\n%s", out)
	}

	output := string(out)
	if !strings.Contains(output, "unable to create change reader for ledger 2") {
		t.Fatalf("expected change reader failure for the first requested ledger, got:\n%s", output)
	}
	if !strings.Contains(output, "requested sequence beyond current LedgerRange") {
		t.Fatalf("expected backend range error from preparing [start,0], got:\n%s", output)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: omitting `--end-ledger` should prepare an unbounded backend range and keep the command alive so it can export newly closed ledgers.
- **Actual**: the command prepares a bounded `[start,0]` range, so the first requested ledger is outside the prepared window and the process exits immediately.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC invokes the real CLI command path with the documented continuous-mode flag shape and observes the first ledger read fail.
2. Realistic preconditions: YES — omitting `--end-ledger` is the documented production interface for continuous export, and the reproduced path uses the default datastore backend rather than a mock.
3. Bug vs by-design: BUG — the command help promises continuous streaming, and sibling code in `PrepareCaptiveCore()` already uses `UnboundedRange(start)` for `end == 0`.
4. Final severity: Medium — this is a concrete operational correctness failure that aborts the export path, but it does not silently corrupt emitted row values.
5. In scope: YES — it is a production CLI path that reliably fails under a documented mode.
6. Test correctness: CORRECT — the final PoC executes the shipping command and checks for the exact backend error caused by the malformed prepared range.
7. Alternative explanations: NONE — the observed error string comes from the buffered backend's `sequence > to` guard, which is exactly the consequence of preparing `[start,0]`.
8. Novelty: NOVEL

## Suggested Fix

Choose the backend range before calling `PrepareRange()`: start with `ledgerbackend.UnboundedRange(startNum)` and only use `ledgerbackend.BoundedRange(startNum, commonArgs.EndNum)` when `commonArgs.EndNum != 0`. Do not rely on rewriting `EndNum` after preparation to simulate streaming mode.
