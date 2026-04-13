# H005: GCS uploads leak source file descriptors and can drop later batch uploads

**Date**: 2026-04-13
**Subsystem**: external-io
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each call to `MaybeUpload()` / `GCS.UploadTo()` should close the local source file after `io.Copy()` finishes so the number of live descriptors stays bounded across long-running or multi-batch exports.

## Mechanism

`GCS.UploadTo()` opens the local file with `os.Open(path)` and never closes the returned reader. Because the function also deletes the path afterward, the leak is easy to miss on Unix: the pathname disappears, but the file descriptor stays live until GC closes it. In continuous or batched exports with cloud upload enabled, those leaked descriptors can accumulate until later uploads fail to open their local files or other export files, causing missing downstream artifacts.

## Trigger

Run a long-lived export with `--cloud-provider gcp`, especially `export_ledger_entry_changes` over many batches or continuous mode (`end-ledger=0`), so the process performs many uploads in a single lifetime.

## Target Code

- `cmd/upload_to_gcs.go:32-35` — opens the source file with `os.Open(path)` and keeps the handle alive
- `cmd/upload_to_gcs.go:52-73` — copies, verifies, deletes the local path, and returns without `reader.Close()`
- `cmd/command_utils.go:123-146` — dispatches uploads whenever cloud output is enabled
- `cmd/export_ledger_entry_changes.go:368-372` — a high-frequency caller that can upload many files in one process

## Evidence

There is no `defer reader.Close()` or any other close path in `UploadTo()`. The prior external-io records already established that missing closes on export files can exhaust file descriptors under sustained operation; this upload helper introduces the same class of leak on the cloud-upload side for every command that calls `MaybeUpload()`.

## Anti-Evidence

Short one-shot commands may exit before the leak becomes user-visible, and Go's GC may eventually reclaim some descriptors. The bug needs a process that performs many uploads in one lifetime, so it is strongest on continuous or highly batched export paths rather than a single small export.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-13
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

`GCS.UploadTo()` opens a local file with `os.Open(path)` at line 32 of `upload_to_gcs.go` and stores the result in `reader`. There is no `defer reader.Close()` and no explicit `reader.Close()` call anywhere in the function. Every code path after a successful `os.Open` — including the happy path (line 73), `storage.NewClient` failure (line 40), `io.Copy` failure (line 54), `wc.Close()` failure (line 58), and `Attrs` verification failure (line 66) — returns without closing the reader. This leaks one file descriptor per upload. All 10 export commands call `MaybeUpload()`, with `export_ledger_entry_changes` being the highest-frequency caller (up to 2 uploads per resource type per batch × 10 resource types).

### Code Paths Examined

- `cmd/upload_to_gcs.go:32` — `reader, err := os.Open(path)` opens file, no subsequent `reader.Close()` anywhere in function
- `cmd/upload_to_gcs.go:40-41` — early return on `storage.NewClient` error leaks reader
- `cmd/upload_to_gcs.go:53-55` — early return on `io.Copy` error leaks reader
- `cmd/upload_to_gcs.go:56-59` — early return on `wc.Close()` error leaks reader
- `cmd/upload_to_gcs.go:64-67` — early return on `Attrs` verification error leaks reader
- `cmd/upload_to_gcs.go:71` — `deleteLocalFiles(path)` removes the filesystem entry but does not close the file descriptor
- `cmd/upload_to_gcs.go:73` — happy-path return without closing reader
- `cmd/command_utils.go:123-146` — `MaybeUpload()` dispatches to `UploadTo()` for every GCS upload
- `cmd/export_ledger_entry_changes.go:368,372` — calls `MaybeUpload` twice per resource (JSON + Parquet) per batch

### Findings

The bug is confirmed. `UploadTo()` has zero close paths for the `reader` variable. This is distinct from the already-confirmed success/external-io/004 finding (which is about `exportTransformedData()` never closing `MustOutFile` JSON output handles). H005 targets a different file handle — the `os.Open()` reader inside the upload function itself.

On Unix, `deleteLocalFiles(path)` removes the directory entry but the file data persists until the last file descriptor is closed. So each upload leaks both the file descriptor and the underlying file data (inode stays allocated). Under sustained operation with GCS uploads enabled, this accumulates one leaked FD per upload across all export commands. The `export_ledger_entry_changes` command is the worst case: with 10 resource types and both JSON + Parquet enabled, each batch leaks up to 20 file descriptors. Over hundreds of batches in continuous mode, this can exhaust the process's file descriptor limit.

### PoC Guidance

- **Test file**: `cmd/upload_to_gcs_test.go` (new file, or append to existing test file in `cmd/`)
- **Setup**: Create a temp directory, write N small files. Disable GC with `debug.SetGCPercent(-1)`. Record FD count before test. Create a mock or stub `UploadTo` variant that performs just the `os.Open` + `io.Copy` to a local sink (to avoid needing real GCS credentials).
- **Steps**: Call the relevant code path N times (e.g., 50 iterations), each opening a distinct file via `os.Open` without closing, then copying to `/dev/null` or a temp writer.
- **Assertion**: After N iterations, measure the next available FD. Assert that `fdAfter - fdBefore >= N`, confirming N file descriptors were leaked. A fix adding `defer reader.Close()` after the `os.Open` error check should reduce the leaked count to 0.
- **Alternative approach**: Since `UploadTo` requires a real GCS client, a simpler PoC could extract the `os.Open` + no-close pattern into a minimal reproducer function that mimics the same code structure, demonstrating the leak without cloud dependencies.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-13
**PoC by**: claude-opus-4.6, high
**Target Test File**: cmd/upload_to_gcs_poc_test.go
**Test Name**: "TestUploadToLeaksFileDescriptors"
**Test Language**: Go

### Demonstration

The test reproduces the exact `os.Open` + `io.Copy` pattern from `GCS.UploadTo()` (lines 32–54 of `upload_to_gcs.go`) with GC disabled to prevent finalizer-based cleanup. After 50 iterations without calling `reader.Close()`, the next available file descriptor number increased by exactly 50, confirming that every call leaks one FD. This proves that `UploadTo()` — which has no `Close()` call on any code path — will exhaust file descriptors under sustained GCS upload workloads.

### Test Body

```go
package cmd

import (
	"fmt"
	"io"
	"os"
	"path/filepath"
	"runtime/debug"
	"testing"
)

// nextFD returns the file descriptor number of the next available FD by
// opening /dev/null and immediately closing it.
func nextFD(t *testing.T) int {
	t.Helper()
	f, err := os.Open("/dev/null")
	if err != nil {
		t.Fatalf("probe fd: %v", err)
	}
	fd := int(f.Fd())
	f.Close()
	return fd
}

// TestUploadToLeaksFileDescriptors demonstrates that the os.Open + io.Copy
// pattern used in GCS.UploadTo() leaks file descriptors because the opened
// reader is never closed.
func TestUploadToLeaksFileDescriptors(t *testing.T) {
	// Disable GC so finalizers cannot reclaim leaked file handles.
	debug.SetGCPercent(-1)
	defer debug.SetGCPercent(100)

	const N = 50
	dir := t.TempDir()

	// Create N small files to simulate upload sources.
	for i := 0; i < N; i++ {
		p := filepath.Join(dir, fmt.Sprintf("upload-%d.txt", i))
		if err := os.WriteFile(p, []byte("payload"), 0644); err != nil {
			t.Fatalf("setup: %v", err)
		}
	}

	fdBefore := nextFD(t)

	// Reproduce the exact pattern from upload_to_gcs.go lines 32-54:
	//   reader, err := os.Open(path)  // line 32
	//   ...
	//   io.Copy(wc, reader)           // line 53
	//   // NO reader.Close() anywhere
	for i := 0; i < N; i++ {
		p := filepath.Join(dir, fmt.Sprintf("upload-%d.txt", i))

		reader, err := os.Open(p)
		if err != nil {
			t.Fatalf("open: %v", err)
		}

		// Simulate the io.Copy to a GCS writer — here we use Discard.
		if _, err := io.Copy(io.Discard, reader); err != nil {
			t.Fatalf("copy: %v", err)
		}

		// BUG: no reader.Close() — matches UploadTo which never closes
		// the reader on any code path.
		_ = reader // keep reference alive so GC cannot finalize
	}

	fdAfter := nextFD(t)
	leaked := fdAfter - fdBefore

	t.Logf("FDs before probe: %d, after probe: %d, leaked: %d (expected >= %d)", fdBefore, fdAfter, leaked, N)

	if leaked < N {
		t.Errorf("Expected at least %d leaked file descriptors, but only %d were leaked — "+
			"hypothesis not demonstrated", N, leaked)
	}
	// If we reach here without the error above, the leak is confirmed:
	// N calls to os.Open without Close produced at least N leaked FDs.
}
```

### Test Output

```
=== RUN   TestUploadToLeaksFileDescriptors
    upload_to_gcs_poc_test.go:72: FDs before probe: 4, after probe: 54, leaked: 50 (expected >= 50)
--- PASS: TestUploadToLeaksFileDescriptors (0.01s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	5.737s
```
