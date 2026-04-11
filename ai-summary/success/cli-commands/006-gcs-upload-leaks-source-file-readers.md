# 006: `UploadTo` leaves local upload readers open until GC

**Date**: 2026-04-11
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`GCS.UploadTo()` opens the local file it is about to upload and never closes that reader. Repeated cloud uploads therefore leave source-file descriptors open until Go's GC/finalizer happens to reclaim them, so long-running export processes can hit the OS file-descriptor limit and stop producing later outputs.

## Root Cause

`UploadTo()` calls `os.Open(path)` to obtain the local upload source, but the function has no `reader.Close()` on either the success path or any error path. The `*os.File` eventually becomes unreachable and may be closed by the Go runtime finalizer, but that cleanup is nondeterministic and happens only after GC, not when the upload completes.

## Reproduction

This code path is exercised during normal operation whenever any export command runs with `--cloud-provider gcp` and uploads a generated JSON or Parquet artifact. The reproduced PoC forces `storage.NewClient()` to fail after the local file is opened, then shows that each `UploadTo()` call consumes one additional live descriptor before GC runs.

## Affected Code

- `cmd/upload_to_gcs.go:UploadTo:25-73` — opens the local upload source with `os.Open(path)` and never closes it
- `cmd/command_utils.go:MaybeUpload:123-146` — invokes `UploadTo()` for each requested cloud upload
- `cmd/export_ledger_entry_changes.go:exportTransformedData:368-372` — repeatedly uploads per-resource JSON and optional Parquet artifacts in batched exports

## PoC

- **Target test file**: `cmd/upload_to_gcs_test.go`
- **Test name**: `TestUploadToLeaksFileDescriptors`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/upload_to_gcs_test.go` with the test body below.
  3. Run `go test ./cmd/... -run TestUploadToLeaksFileDescriptors -v`
  4. Observe the next available descriptor number increase by one per upload attempt.

### Test Body

```go
package cmd

import (
	"fmt"
	"os"
	"path/filepath"
	"runtime"
	"testing"
)

// nextAvailableFD opens /dev/null to observe the next descriptor number the OS
// would hand out, then closes it immediately.
func nextAvailableFD(t *testing.T) int {
	t.Helper()

	f, err := os.Open(os.DevNull)
	if err != nil {
		t.Fatalf("open %s: %v", os.DevNull, err)
	}
	defer f.Close()

	return int(f.Fd())
}

func TestUploadToLeaksFileDescriptors(t *testing.T) {
	runtime.GC()

	dir := t.TempDir()
	const uploads = 20
	files := make([]string, 0, uploads)
	for i := 0; i < uploads; i++ {
		path := filepath.Join(dir, fmt.Sprintf("upload-%02d.txt", i))
		if err := os.WriteFile(path, []byte("payload"), 0o644); err != nil {
			t.Fatalf("write %s: %v", path, err)
		}
		files = append(files, path)
	}

	before := nextAvailableFD(t)

	gcs := &GCS{}
	for _, path := range files {
		err := gcs.UploadTo("/nonexistent/invalid-credentials.json", "fake-bucket", path)
		if err == nil {
			t.Fatalf("UploadTo(%q) unexpectedly succeeded", path)
		}
	}

	after := nextAvailableFD(t)
	leaked := after - before
	t.Logf("next available fd before=%d after=%d leaked=%d", before, after, leaked)

	if leaked < uploads {
		t.Fatalf("expected at least %d leaked descriptors, got %d", uploads, leaked)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each upload should close its local source file before returning, so repeated uploads keep the number of live descriptors bounded.
- **Actual**: Each upload attempt leaves its source reader open until a later GC/finalizer pass, so the process accumulates open descriptors across uploads.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC calls the real `GCS.UploadTo()` implementation and measures descriptor growth after repeated invocations.
2. Realistic preconditions: YES — this is the normal upload path used by CLI exports when GCS upload is enabled.
3. Bug vs by-design: BUG — `UploadTo()` owns the file handle it opens and should close it deterministically rather than relying on GC.
4. Final severity: Medium — this is operational correctness breakage that can stop later uploads/exports under sustained normal operation.
5. In scope: YES — the objective explicitly includes resource leaks that cause data loss under sustained operation.
6. Test correctness: CORRECT — the test uses production code, a realistic local file input, and an observable OS-level effect instead of a tautological assertion.
7. Alternative explanations: Go finalizers can eventually close unreachable `*os.File` values, but that only happens after GC; the reproduced descriptor growth before GC is the bug and is sufficient to exhaust process FDs in long-running jobs.
8. Novelty: NOVEL

## Suggested Fix

Add `defer reader.Close()` immediately after `os.Open(path)` succeeds in `UploadTo()`, so both success and error paths release the source file descriptor deterministically.
