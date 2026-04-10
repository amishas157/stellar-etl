# 001: Parquet Upload Before Write Breaks Parquet Export

**Date**: 2026-04-10
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

Four CLI export commands call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. When `--write-parquet` and GCS upload are both enabled, the code attempts to open the parquet file before that file exists, so the run can upload JSON first and then fatally fail before generating or uploading the current parquet output.

## Root Cause

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` use the wrong parquet lifecycle order. `MaybeUpload` delegates to `GCS.UploadTo`, and `UploadTo` immediately calls `os.Open(path)`, so the parquet upload step requires the local parquet file to already exist.

## Reproduction

Run one of the affected commands with `--write-parquet` and `--cloud-provider gcp`. The command closes and uploads the JSON output first, then enters the parquet block, where `MaybeUpload(parquetPath)` executes before `WriteParquet(...)`; `UploadTo` fails with `failed to open file ...`, `MaybeUpload` fatals, and the current parquet file is never written.

## Affected Code

- `cmd/export_ledgers.go:Run:68-72` — uploads `parquetPath` before writing it
- `cmd/export_transactions.go:Run:61-65` — uploads `parquetPath` before writing it
- `cmd/export_operations.go:Run:61-65` — uploads `parquetPath` before writing it
- `cmd/export_trades.go:Run:66-70` — uploads `parquetPath` before writing it
- `cmd/command_utils.go:MaybeUpload:123-146` — fatals on upload error
- `cmd/upload_to_gcs.go:GCS.UploadTo:25-35` — opens the local file before any remote upload work starts

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestParquetUploadPrecedesWrite` and `TestUploadToFailsWhenParquetFileNotYetWritten`
- **Test language**: go
- **How to run**: Create `cmd/data_integrity_poc_test.go` with the test body below, then run `go test ./cmd/... -run 'TestParquetUploadPrecedesWrite|TestUploadToFailsWhenParquetFileNotYetWritten' -v`.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"runtime"
	"strings"
	"testing"
)

func sourceDir() string {
	_, filename, _, _ := runtime.Caller(0)
	return filepath.Dir(filename)
}

func TestParquetUploadPrecedesWrite(t *testing.T) {
	type testCase struct {
		file              string
		expectUploadFirst bool
	}

	cases := []testCase{
		{file: "export_ledgers.go", expectUploadFirst: true},
		{file: "export_transactions.go", expectUploadFirst: true},
		{file: "export_operations.go", expectUploadFirst: true},
		{file: "export_trades.go", expectUploadFirst: true},
		{file: "export_effects.go", expectUploadFirst: false},
		{file: "export_assets.go", expectUploadFirst: false},
	}

	for _, tc := range cases {
		t.Run(tc.file, func(t *testing.T) {
			content, err := os.ReadFile(filepath.Join(sourceDir(), tc.file))
			if err != nil {
				t.Fatalf("failed to read %s: %v", tc.file, err)
			}

			lines := strings.Split(string(content), "\n")
			maybeUploadLine := -1
			writeParquetLine := -1

			for i := 0; i < len(lines); i++ {
				line := strings.TrimSpace(lines[i])
				if !strings.HasPrefix(line, "if commonArgs.WriteParquet") {
					continue
				}

				blockUploadLine := -1
				blockWriteLine := -1
				for j := i + 1; j < len(lines); j++ {
					inner := strings.TrimSpace(lines[j])
					if strings.Contains(inner, "MaybeUpload(") && blockUploadLine == -1 {
						blockUploadLine = j + 1
					}
					if strings.Contains(inner, "WriteParquet(") && blockWriteLine == -1 {
						blockWriteLine = j + 1
					}
					if inner == "}" {
						break
					}
				}

				if blockUploadLine != -1 && blockWriteLine != -1 {
					maybeUploadLine = blockUploadLine
					writeParquetLine = blockWriteLine
					break
				}
			}

			if maybeUploadLine == -1 || writeParquetLine == -1 {
				t.Fatalf("failed to find parquet upload/write block in %s", tc.file)
			}

			uploadFirst := maybeUploadLine < writeParquetLine
			if uploadFirst != tc.expectUploadFirst {
				t.Fatalf(
					"unexpected order in %s: MaybeUpload=%d WriteParquet=%d wantUploadFirst=%t",
					tc.file,
					maybeUploadLine,
					writeParquetLine,
					tc.expectUploadFirst,
				)
			}
		})
	}
}

func TestUploadToFailsWhenParquetFileNotYetWritten(t *testing.T) {
	gcs := &GCS{gcsCredentialsPath: "", gcsBucket: "test-bucket"}
	parquetPath := filepath.Join(t.TempDir(), "not-yet-written.parquet")

	err := gcs.UploadTo("", "test-bucket", parquetPath)
	if err == nil {
		t.Fatal("expected upload to fail for a parquet file that has not been written yet")
	}

	if !strings.Contains(err.Error(), "failed to open file") {
		t.Fatalf("expected open-file failure, got %v", err)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: The exporters should write the parquet file first and only then upload that freshly generated file.
- **Actual**: The affected exporters attempt the parquet upload first, so the upload step runs against a file that has not yet been generated for the current export.

## Adversarial Review

1. Exercises claimed bug: YES — one test proves the exact command ordering in the affected exporters, and the other proves that the upload path fails when that file has not been written yet.
2. Realistic preconditions: YES — `--write-parquet` and `--cloud-provider gcp` are supported runtime flags, and `parquet-output` is a distinct path.
3. Bug vs by-design: BUG — sibling exporters in the same package use the opposite order (`WriteParquet` then `MaybeUpload`), which matches the only workable lifecycle.
4. Final severity: Medium — the reproducible impact is an output-ordering failure that can leave JSON uploaded while the current parquet export is missing; this is operational correctness, not a proven high-severity data remapping issue.
5. In scope: YES — it affects correctness of exported artifacts in normal supported CLI usage.
6. Test correctness: CORRECT — the tests use production source and production upload logic, not a tautological mock.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Swap the parquet operations in the four affected commands so `WriteParquet(...)` always runs before `MaybeUpload(..., parquetPath)`.
