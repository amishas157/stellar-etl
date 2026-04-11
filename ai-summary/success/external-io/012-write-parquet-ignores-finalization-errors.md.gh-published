# 012: `WriteParquet()` ignores finalization errors and returns success on corrupt parquet

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`cmd.WriteParquet()` checks `writer.Write()` errors for individual records, but it defers both `writer.WriteStop()` and `parquetFile.Close()` without inspecting either return value. When the underlying file accepts all earlier writes and then fails during the final footer write, `WriteParquet()` still returns success and leaves a malformed parquet file behind for callers to keep or upload.

The original PoC only reproduced the defer pattern in a helper function. I rewrote it into a self-contained test that calls the production `WriteParquet()` path and runs it in a subprocess with a tight file-size limit, which reliably simulates a late disk-full error after row writes have already been accepted.

## Root Cause

The parquet-go writer does its final flush and footer emission inside `WriteStop()`, and that method explicitly returns any write error from flushing row groups, writing indexes, writing the footer size, or writing the trailing `PAR1` magic bytes. `WriteParquet()` discards that return value with `defer writer.WriteStop()`, so late filesystem failures are silently lost even though the file is not a valid parquet artifact.

## Reproduction

During normal operation, this path is reachable whenever the destination file starts failing only at the end of the export lifecycle — for example, when a filesystem runs out of space during the final footer write or reports a late writeback/finalization error after earlier data pages were accepted. In that situation, `WriteParquet()` returns normally, the caller sees no failure, and commands that call `MaybeUpload()` afterward can propagate the corrupt artifact.

## Affected Code

- `cmd/command_utils.go:162-180` — `WriteParquet()` defers `parquetFile.Close()` and `writer.WriteStop()` and ignores both return values
- `cmd/export_assets.go:80-81` — uploads immediately after `WriteParquet()` returns
- `cmd/export_effects.go:67-68` — uploads immediately after `WriteParquet()` returns
- `cmd/export_contract_events.go:64-65` — uploads immediately after `WriteParquet()` returns
- `cmd/export_ledger_entry_changes.go:371-372` — uploads immediately after `WriteParquet()` returns
- `github.com/xitongsys/parquet-go/writer.(*ParquetWriter).WriteStop:128-202` — returns errors from final flush, footer, footer-size, and trailing magic writes

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestWriteParquetIgnoresFinalizationErrors`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./cmd/... -run TestWriteParquetIgnoresFinalizationErrors -v`.

### Test Body

```go
package cmd

import (
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

type writeParquetPoCInput struct {
	Name  string
	Value int32
}

type writeParquetPoCRecord struct {
	Name  string `parquet:"name=name, type=BYTE_ARRAY, convertedtype=UTF8"`
	Value int32  `parquet:"name=value, type=INT32"`
}

func (r writeParquetPoCInput) ToParquet() interface{} {
	return writeParquetPoCRecord{
		Name:  r.Name,
		Value: r.Value,
	}
}

func TestWriteParquetIgnoresFinalizationErrors(t *testing.T) {
	records := []transform.SchemaParquet{
		writeParquetPoCInput{Name: "late-write", Value: 42},
	}

	tempDir := t.TempDir()
	validPath := filepath.Join(tempDir, "valid.parquet")
	WriteParquet(records, validPath, new(writeParquetPoCRecord))

	validBytes, err := os.ReadFile(validPath)
	if err != nil {
		t.Fatalf("read valid parquet: %v", err)
	}
	if !hasParquetFooter(validBytes) {
		t.Fatalf("baseline parquet missing footer (%d bytes)", len(validBytes))
	}

	corruptPath := filepath.Join(tempDir, "corrupt.parquet")
	fullSize := len(validBytes)
	helperPath := buildWriteParquetHelper(t)

	for limit := fullSize - 1; limit > 0; limit-- {
		_ = os.Remove(corruptPath)

		cmd := exec.Command(helperPath)
		cmd.Env = append(
			os.Environ(),
			"STELLAR_ETL_PARQUET_LIMIT="+strconv.Itoa(limit),
			"STELLAR_ETL_PARQUET_PATH="+corruptPath,
		)

		if output, err := cmd.CombinedOutput(); err != nil {
			t.Logf("limit=%d exited before returning from WriteParquet: %v\n%s", limit, err, output)
			continue
		}

		corruptBytes, err := os.ReadFile(corruptPath)
		if err != nil {
			t.Fatalf("read corrupt parquet at limit %d: %v", limit, err)
		}
		if hasParquetFooter(corruptBytes) {
			continue
		}

		t.Logf(
			"WriteParquet returned success at file size limit %d, but produced a %d-byte file without the required PAR1 footer (valid file is %d bytes)",
			limit,
			len(corruptBytes),
			fullSize,
		)
		return
	}

	t.Fatalf("could not find a file size limit below %d bytes that let WriteParquet return success while omitting the footer", fullSize)
}

func buildWriteParquetHelper(t *testing.T) string {
	t.Helper()

	root := repoRoot(t)
	tempDir, err := os.MkdirTemp(root, ".write-parquet-helper-*")
	if err != nil {
		t.Fatalf("create helper temp dir: %v", err)
	}
	t.Cleanup(func() {
		_ = os.RemoveAll(tempDir)
	})
	sourcePath := filepath.Join(tempDir, "main.go")
	binaryPath := filepath.Join(tempDir, "write-parquet-helper")
	if err := os.WriteFile(sourcePath, []byte(writeParquetHelperSource), 0o600); err != nil {
		t.Fatalf("write helper source: %v", err)
	}

	buildCmd := exec.Command("go", "build", "-o", binaryPath, sourcePath)
	buildCmd.Dir = root
	if output, err := buildCmd.CombinedOutput(); err != nil {
		t.Fatalf("build helper: %v\n%s", err, output)
	}

	return binaryPath
}

func hasParquetFooter(data []byte) bool {
	return len(data) >= 4 && string(data[len(data)-4:]) == "PAR1"
}

func repoRoot(t *testing.T) string {
	t.Helper()

	dir, err := os.Getwd()
	if err != nil {
		t.Fatalf("get working directory: %v", err)
	}

	for {
		if _, err := os.Stat(filepath.Join(dir, "go.mod")); err == nil {
			return dir
		}

		parent := filepath.Dir(dir)
		if parent == dir {
			t.Fatal("could not locate repository root")
		}
		dir = parent
	}
}

const writeParquetHelperSource = `package main

import (
	"os"
	"os/signal"
	"strconv"
	"syscall"

	etlcmd "github.com/stellar/stellar-etl/v2/cmd"
	"github.com/stellar/stellar-etl/v2/internal/transform"
)

type writeParquetPoCInput struct {
	Name  string
	Value int32
}

type writeParquetPoCRecord struct {
	Name  string ` + "`parquet:\"name=name, type=BYTE_ARRAY, convertedtype=UTF8\"`" + `
	Value int32  ` + "`parquet:\"name=value, type=INT32\"`" + `
}

func (r writeParquetPoCInput) ToParquet() interface{} {
	return writeParquetPoCRecord{
		Name:  r.Name,
		Value: r.Value,
	}
}

func main() {
	limit, err := strconv.ParseUint(os.Getenv("STELLAR_ETL_PARQUET_LIMIT"), 10, 64)
	if err != nil {
		panic(err)
	}
	path := os.Getenv("STELLAR_ETL_PARQUET_PATH")
	if path == "" {
		panic("missing parquet path")
	}

	signal.Ignore(syscall.SIGXFSZ)
	if err := syscall.Setrlimit(syscall.RLIMIT_FSIZE, &syscall.Rlimit{Cur: limit, Max: limit}); err != nil {
		panic(err)
	}

	etlcmd.WriteParquet(
		[]transform.SchemaParquet{
			writeParquetPoCInput{Name: "late-write", Value: 42},
		},
		path,
		new(writeParquetPoCRecord),
	)
}
`
```

## Expected vs Actual Behavior

- **Expected**: a late failure during parquet finalization should be surfaced so the export command aborts and does not keep or upload a malformed parquet file.
- **Actual**: `WriteParquet()` returns success after `WriteStop()` fails, and callers observe a truncated file that is missing the required `PAR1` footer.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC calls the production `WriteParquet()` helper and demonstrates that it exits successfully while leaving a malformed file behind.
2. Realistic preconditions: YES — a file-size cap is a faithful stand-in for a destination that begins rejecting writes only during the final flush/footer stage.
3. Bug vs by-design: BUG — parquet-go documents `WriteStop()` as the error-reporting finalization step, and its own example code checks that error before declaring success.
4. Final severity: Medium — the exported parquet payload becomes silently unusable, but the corruption is an operational export failure rather than a wrong financial field value.
5. In scope: YES — this is a concrete silent-failure path in the export pipeline.
6. Test correctness: CORRECT — the test first establishes a valid baseline parquet file, then reproduces the failure on the real helper and verifies the malformed output by checking the missing footer.
7. Alternative explanations: NONE — the subprocess returns success, so the malformed file cannot be explained by early row-write failure or an aborted process.
8. Novelty: NOVEL

## Suggested Fix

Change `WriteParquet()` to stop deferring unchecked finalization and instead return an `error`. Call `writer.WriteStop()` explicitly, check and return that error, then close the file and propagate any close failure before any caller invokes `MaybeUpload()`.
