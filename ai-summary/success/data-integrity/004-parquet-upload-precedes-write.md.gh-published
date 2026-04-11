# 004: Parquet Upload Precedes Write

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

Four one-shot export commands call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. On a fresh path this fails at `os.Open(path)` before the current parquet file exists; on a reused path it can upload the previous run's parquet object, delete the local copy, and only then write the current run's parquet file locally.

## Root Cause

`export_transactions`, `export_operations`, `export_ledgers`, and `export_trades` reverse the output lifecycle for parquet artifacts. `GCS.UploadTo()` requires the local file to already exist and deletes it after a successful upload, but `WriteParquet()` is the first code that materializes the current parquet file.

## Reproduction

Run any affected exporter with both `--write-parquet` and `--cloud-provider gcp`. The normal command path finishes JSON export, uploads JSON, then tries to upload the parquet path before generating the new parquet file; because `parquet-output` defaults to a stable filename, repeated runs naturally reuse the same local parquet path.

## Affected Code

- `cmd/export_transactions.go:63-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_operations.go:63-65` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_ledgers.go:70-72` — uploads `parquetPath` before `WriteParquet`
- `cmd/export_trades.go:68-70` — uploads `parquetPath` before `WriteParquet`
- `cmd/upload_to_gcs.go:25-35` — `UploadTo()` immediately opens the local path for reading
- `cmd/upload_to_gcs.go:69-71` — successful upload deletes the local file
- `internal/utils/main.go:250-253` — archive exporters reuse a stable default `parquet-output` filename across runs

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestParquetUploadPrecedesWrite`
- **Test language**: go
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create test file at `cmd/data_integrity_poc_test.go`
  3. Run: `go test ./cmd/... -run TestParquetUploadPrecedesWrite -v`
  4. Observe: the test proves the affected exporters place parquet upload before parquet generation, and the production uploader fails on a fresh parquet path because the file does not exist yet.

### Test Body

```go
package cmd

import (
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

func readSource(t *testing.T, relativePath string) string {
	t.Helper()

	content, err := os.ReadFile(relativePath)
	if err != nil {
		t.Fatalf("failed to read %s: %v", relativePath, err)
	}

	return string(content)
}

func requireOrder(t *testing.T, source, first, second, context string) {
	t.Helper()

	firstIndex := strings.Index(source, first)
	if firstIndex == -1 {
		t.Fatalf("missing %q in %s", first, context)
	}

	secondIndex := strings.Index(source, second)
	if secondIndex == -1 {
		t.Fatalf("missing %q in %s", second, context)
	}

	if firstIndex >= secondIndex {
		t.Fatalf("unexpected order in %s: %q should appear before %q", context, first, second)
	}
}

func TestParquetUploadPrecedesWrite(t *testing.T) {
	t.Run("affected export commands upload parquet before writing it", func(t *testing.T) {
		for _, relativePath := range []string{
			"cmd/export_transactions.go",
			"cmd/export_operations.go",
			"cmd/export_ledgers.go",
			"cmd/export_trades.go",
		} {
			source := readSource(t, relativePath)
			requireOrder(
				t,
				source,
				"MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)",
				"WriteParquet(",
				relativePath,
			)
		}
	})

	t.Run("sibling commands use the safe order", func(t *testing.T) {
		for _, relativePath := range []string{
			"cmd/export_effects.go",
			"cmd/export_assets.go",
		} {
			source := readSource(t, relativePath)
			requireOrder(
				t,
				source,
				"WriteParquet(",
				"MaybeUpload(cloudCredentials, cloudStorageBucket, cloudProvider, parquetPath)",
				relativePath,
			)
		}
	})

	t.Run("uploading a parquet path before writing it fails on a fresh run", func(t *testing.T) {
		tmpDir := t.TempDir()
		parquetPath := filepath.Join(tmpDir, "exported_transactions.parquet")

		err := (&GCS{}).UploadTo("", "fake-bucket", parquetPath)
		if err == nil {
			t.Fatal("expected UploadTo to fail for a parquet path that has not been written yet")
		}
		if !strings.Contains(err.Error(), "failed to open file") {
			t.Fatalf("expected missing-file failure, got %v", err)
		}

		var records []transform.SchemaParquet
		WriteParquet(records, parquetPath, new(transform.TransactionOutputParquet))

		if _, err := os.Stat(parquetPath); err != nil {
			t.Fatalf("expected WriteParquet to create %s: %v", parquetPath, err)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: the exporter writes the current parquet artifact first and then uploads that same file.
- **Actual**: the affected commands attempt upload first; fresh runs fail before parquet generation, while reused paths can upload a stale prior-run parquet object and only regenerate the current file locally afterward.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC verifies the exact source ordering in the affected commands and separately executes the production `GCS.UploadTo()` / `WriteParquet()` behavior that makes the ordering incorrect.
2. Realistic preconditions: YES — `--write-parquet` and cloud upload are first-class CLI flags, and the default `parquet-output` path is stable across runs.
3. Bug vs by-design: BUG — sibling exporters already use write-then-upload, and `UploadTo()` semantics require that order.
4. Final severity: Medium — this breaks or stales the parquet artifact lifecycle, but it is not direct monetary field corruption.
5. In scope: YES — it causes exporter data loss / stale remote data under a supported output mode.
6. Test correctness: CORRECT — it is not tautological; it checks shipped source order and a real runtime failure in the production uploader.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Swap the order in the four affected exporters so `WriteParquet(...)` completes successfully before `MaybeUpload(..., parquetPath)` runs, matching the existing `export_effects` and `export_assets` pattern.
