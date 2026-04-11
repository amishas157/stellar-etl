# 006: Token-transfer limit counts ledgers instead of token-transfer rows

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Operational correctness: `--limit` can return empty or oversized token-transfer exports
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`export_token_transfer --limit N` is documented as the maximum number of token-transfer rows to export, but the command applies that budget to source ledgers before `TransformTokenTransfer()` expands each ledger into rows. On pubnet, `export_token_transfer -s 30822015 -e 30822025 -l 1` writes `57` rows, and `export_token_transfer -s 10363513 -e 10363514 -l 1` writes `0` rows even though ledger `10363514` contains a token-transfer row.

## Root Cause

`cmd/export_token_transfers.go` passes the caller-visible `limit` directly into `input.GetLedgers()`, and `GetLedgers()` enforces that limit on `len(ledgerSlice)`. `transform.TransformTokenTransfer()` then expands each returned ledger into all token-transfer events from that ledger, but `export_token_transfer` never applies a secondary cap after that expansion.

## Reproduction

During normal operation, this manifests whenever the first limited ledger emits anything other than exactly one token-transfer row. Real pubnet reproductions are:

```bash
./stellar-etl export_token_transfer -s 30822015 -e 30822025 -l 1 -o /tmp/token-transfer-limited.json
wc -l /tmp/token-transfer-limited.json

./stellar-etl export_token_transfer -s 10363513 -e 10363514 -l 1 -o /tmp/token-transfer-empty.json
wc -l /tmp/token-transfer-empty.json

./stellar-etl export_token_transfer -s 10363514 -e 10363514 -o /tmp/token-transfer-next-ledger.json
wc -l /tmp/token-transfer-next-ledger.json
```

The first command writes `57` rows despite `-l 1`. The second command writes `0` rows while logging one successful transform, and the third command shows the skipped next ledger contains `1` token-transfer row.

## Affected Code

- `cmd/export_token_transfers.go:21-54` — passes `limit` into `GetLedgers()` and then exports every row returned by `TransformTokenTransfer()`
- `internal/input/ledgers.go:14-90` — enforces `limit` on ledger count via `len(ledgerSlice)`
- `internal/transform/token_transfer.go:14-34` — converts each returned ledger into its token-transfer events
- `internal/transform/token_transfer.go:37-129` — expands one ledger into 0..N `TokenTransferOutput` rows
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `token_transfer` objects to export

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTokenTransferLimitCountsLedgersNotRows`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform_test

import (
	"bufio"
	"bytes"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"

	"github.com/stretchr/testify/require"
)

func TestTokenTransferLimitCountsLedgersNotRows(t *testing.T) {
	cwd, err := os.Getwd()
	require.NoError(t, err)

	repoRoot := filepath.Clean(filepath.Join(cwd, "..", ".."))
	tmpDir := t.TempDir()
	executable := filepath.Join(tmpDir, "stellar-etl")

	build := exec.Command("go", "build", "-o", executable, ".")
	build.Dir = repoRoot
	buildOutput, err := build.CombinedOutput()
	require.NoErrorf(t, err, "build failed: %s", buildOutput)

	help := exec.Command(executable, "export_token_transfer", "--help")
	help.Dir = repoRoot
	helpOutput, err := help.CombinedOutput()
	require.NoErrorf(t, err, "help failed: %s", helpOutput)
	require.Contains(t, string(helpOutput), "Maximum number of token_transfer to export")

	limitedOutput := filepath.Join(tmpDir, "limited.txt")
	limited := exec.Command(executable,
		"export_token_transfer",
		"-s", "30822015",
		"-e", "30822025",
		"-l", "1",
		"-o", limitedOutput,
	)
	limited.Dir = repoRoot
	limitedRunOutput, err := limited.CombinedOutput()
	require.NoErrorf(t, err, "limited export failed: %s", limitedRunOutput)

	singleLedgerOutput := filepath.Join(tmpDir, "single-ledger.txt")
	singleLedger := exec.Command(executable,
		"export_token_transfer",
		"-s", "30822015",
		"-e", "30822015",
		"-o", singleLedgerOutput,
	)
	singleLedger.Dir = repoRoot
	singleLedgerRunOutput, err := singleLedger.CombinedOutput()
	require.NoErrorf(t, err, "single-ledger export failed: %s", singleLedgerRunOutput)

	limitedRows := countLines(t, limitedOutput)
	singleLedgerRows := countLines(t, singleLedgerOutput)
	require.Greater(t, limitedRows, 1, "limit=1 should not emit more than one token_transfer row")
	require.Equal(t, singleLedgerRows, limitedRows, "limit=1 over a wider range should match exactly one ledger of output")

	limitedBytes, err := os.ReadFile(limitedOutput)
	require.NoError(t, err)
	singleLedgerBytes, err := os.ReadFile(singleLedgerOutput)
	require.NoError(t, err)
	require.True(t, bytes.Equal(singleLedgerBytes, limitedBytes), "limit=1 should not export an entire first ledger worth of rows")

	emptyLimitedOutput := filepath.Join(tmpDir, "empty-limited.txt")
	emptyLimited := exec.Command(executable,
		"export_token_transfer",
		"-s", "10363513",
		"-e", "10363514",
		"-l", "1",
		"-o", emptyLimitedOutput,
	)
	emptyLimited.Dir = repoRoot
	emptyLimitedRunOutput, err := emptyLimited.CombinedOutput()
	require.NoErrorf(t, err, "empty-first-ledger export failed: %s", emptyLimitedRunOutput)

	nextLedgerOutput := filepath.Join(tmpDir, "next-ledger.txt")
	nextLedger := exec.Command(executable,
		"export_token_transfer",
		"-s", "10363514",
		"-e", "10363514",
		"-o", nextLedgerOutput,
	)
	nextLedger.Dir = repoRoot
	nextLedgerRunOutput, err := nextLedger.CombinedOutput()
	require.NoErrorf(t, err, "next-ledger export failed: %s", nextLedgerRunOutput)

	require.Equal(t, 0, countLines(t, emptyLimitedOutput), "limit=1 should not allow an empty first ledger to consume the entire token_transfer budget")
	require.Greater(t, countLines(t, nextLedgerOutput), 0, "the next ledger should prove token_transfer rows existed within the requested range")
}

func countLines(t *testing.T, path string) int {
	t.Helper()

	file, err := os.Open(path)
	require.NoError(t, err)
	defer file.Close()

	count := 0
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		if strings.TrimSpace(scanner.Text()) != "" {
			count++
		}
	}

	require.NoError(t, scanner.Err())
	return count
}
```

## Expected vs Actual Behavior

- **Expected**: `export_token_transfer --limit N` should stop after `N` token-transfer rows have actually been emitted, matching the flag text.
- **Actual**: `GetLedgers()` stops after `N` ledgers, and `export_token_transfer` then writes every row from those ledgers, so `--limit 1` can emit zero rows or many rows. Verified real outputs are `0` and `57`.

## Adversarial Review

1. Exercises claimed bug: YES — the independent PoC runs the real CLI and shows both failure modes: `-l 1` over `30822015..30822025` writes `57` rows, while `-l 1` over `10363513..10363514` writes `0` rows even though the next ledger has data.
2. Realistic preconditions: YES — ledgers with many token-transfer events and ledgers with no transactions are both normal pubnet conditions.
3. Bug vs by-design: BUG — `AddArchiveFlags("token_transfer", ...)` documents `--limit` as a token-transfer-row budget, but the exporter spends it on ledgers.
4. Final severity: Medium — this silently breaks caller-visible batching semantics and can drop or overshoot exports, but it does not alter financial field values inside emitted rows.
5. In scope: YES — this is a concrete production export path that returns wrong-but-plausible output without error.
6. Test correctness: CORRECT — the PoC does not assume the bug; it compares a limited multi-ledger export against an explicit single-ledger export and separately proves the empty-first-ledger truncation on real archived ledgers.
7. Alternative explanations: NONE — once `GetLedgers(limit=1)` returns exactly one ledger and `TransformTokenTransfer()` expands that ledger into 0..N rows, the current exporter necessarily writes the wrong number of rows because no secondary cap exists.
8. Novelty: NOVEL

## Suggested Fix

Apply `limit` to emitted `TokenTransferOutput` rows rather than to source ledgers. The simplest fix is to track written row count in `export_token_transfers.go` and stop once it reaches the requested bound; alternatively, add a token-transfer-level reader that enforces the limit after ledger expansion.
