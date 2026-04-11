# H001: GCS upload leaks source file descriptors across uploaded exports

**Date**: 2026-04-11
**Subsystem**: cli-commands
**Severity**: Medium
**Impact**: operational correctness
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Each export that uploads a local file to GCS should close the local file reader after `io.Copy` completes so long-running exports can keep opening, uploading, and deleting later output files. A successful upload should not leave an open descriptor behind for the uploaded file.

## Mechanism

`GCS.UploadTo` opens the local file with `os.Open(path)` and streams it into the GCS writer, but never closes that `reader`. Commands that call `MaybeUpload` repeatedly — especially `export_ledger_entry_changes`, which uploads one JSON file and optionally one parquet file per resource per batch — accumulate leaked descriptors until later exports or uploads fail to open their next file, silently dropping later output ranges after earlier uploads appeared successful.

## Trigger

Run any command with `--cloud-provider gcp`, especially `export_ledger_entry_changes` over many batches or a command that uploads both JSON and parquet files in the same process. After enough uploaded files, the process approaches the OS open-file limit because each uploaded file remains open even after `deleteLocalFiles(path)` removes the pathname.

## Target Code

- `cmd/upload_to_gcs.go:25-73` — `UploadTo` opens the local file reader, copies it, and never closes it
- `cmd/command_utils.go:123-146` — `MaybeUpload` invokes `UploadTo` for every uploaded output file
- `cmd/export_ledger_entry_changes.go:368-372` — long-running batch exporter repeatedly uploads per-resource JSON/parquet files
- `cmd/export_assets.go:77-81` — one-shot exporters also hit the same leaked-reader path for both JSON and parquet uploads

## Evidence

The upload path has `reader, err := os.Open(path)` at line 32 and no matching `defer reader.Close()` anywhere in the function. The same function then deletes the uploaded pathname via `deleteLocalFiles(path)` at line 71, so the leak persists only as an unlinked-but-open descriptor that remains invisible to later cleanup.

## Anti-Evidence

Short runs that upload only one or two files may exit before the descriptor leak becomes observable. The corruption mechanism depends on repeated uploads in a single process rather than on any specific ledger content.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `UploadTo` in `cmd/upload_to_gcs.go:25-73`. The function opens a local file at line 32 via `os.Open(path)` assigning to `reader`, then streams it to GCS via `io.Copy` at line 53. There is no `reader.Close()` or `defer reader.Close()` anywhere in the function. The GCS `client` has `defer client.Close()` at line 42, but the local file reader has no equivalent. All five code paths through the function (success + four error returns) leak the reader. This is distinct from the already-confirmed finding 004 which covers `outFile` leaks in `exportTransformedData`.

### Code Paths Examined

- `cmd/upload_to_gcs.go:UploadTo:25-74` — `reader, err := os.Open(path)` at line 32 is never closed; `defer client.Close()` exists at line 42 for the GCS client but no matching close for the reader; `deleteLocalFiles(path)` at line 71 unlinks the file but the open descriptor persists
- `cmd/command_utils.go:MaybeUpload:123-146` — calls `cloudStorage.UploadTo()` at line 138; no post-call cleanup of the leaked reader since it's internal to `UploadTo`
- `cmd/export_ledger_entry_changes.go:exportTransformedData:368-372` — calls `MaybeUpload` up to twice per resource per batch (once for JSON at 368, once for Parquet at 372), making the leak cumulative across batches

### Findings

1. **Missing `reader.Close()`**: `UploadTo` opens `reader` at line 32 and never closes it. The GCS client gets `defer client.Close()` but the reader does not. This is a straightforward omission.
2. **All error paths also leak**: If `storage.NewClient()` fails (line 39), `io.Copy()` fails (line 53), `wc.Close()` fails (line 57), or `pathObj.Attrs()` fails (line 65), the function returns an error without closing `reader`.
3. **Cumulative in batch mode**: `export_ledger_entry_changes` calls `MaybeUpload` up to 2× per resource type per batch. With 10 resource types and Parquet enabled, that's up to 20 leaked descriptors per batch. Over hundreds of batches in continuous mode, this reaches OS limits.
4. **Distinct from finding 004**: Success finding 004 covers `outFile` from `MustOutFile()` not being closed in `exportTransformedData`. This hypothesis covers a separate `reader` in `UploadTo` — different function, different file handle, different fix location. Even if 004 were fixed, this leak would persist.
5. **Invisible after `deleteLocalFiles`**: The file is unlinked at line 71 but the open descriptor keeps the inode allocated in memory. Tools like `ls` won't see it; only `/proc/PID/fd` or `lsof` would reveal the zombie descriptors.

### PoC Guidance

- **Test file**: `cmd/upload_to_gcs_test.go` (new file, or append to existing if present)
- **Setup**: Create a temp directory with several small files. Mock or stub the GCS client (or use a test that counts open FDs before/after calling `UploadTo` with a non-GCS path to trigger the `os.Open` but early-return on client error).
- **Steps**: Call `UploadTo` repeatedly (e.g., 50 times) with valid local files but invalid GCS credentials so the function opens `reader` and returns an error at the `storage.NewClient` step. Count open file descriptors before and after.
- **Assertion**: Assert that the number of open file descriptors grows by at least the number of `UploadTo` calls, demonstrating the reader is never closed.
- **Alternative simpler PoC**: Static analysis — confirm `reader` is assigned at line 32, search for `reader.Close()` in the function body, and assert it does not exist.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4-6, high
**Target Test File**: cmd/upload_to_gcs_test.go
**Test Name**: "TestUploadToLeaksFileDescriptors"
**Test Language**: Go

### Demonstration

The test calls `GCS.UploadTo` 20 times with valid local files but invalid GCS credentials (forcing `storage.NewClient` to fail). Each call opens the local file via `os.Open` at line 32 of `upload_to_gcs.go` but returns an error at line 40 without ever closing the reader. By measuring the next available file descriptor number before and after the 20 calls, the test proves that exactly 20 FDs were leaked — one per `UploadTo` invocation — confirming the missing `reader.Close()`.

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

// nextAvailableFD returns the lowest available file descriptor number by opening
// /dev/null, recording its FD, and immediately closing it. Since the OS assigns
// the lowest unused FD, this serves as a proxy for how many FDs are open.
func nextAvailableFD(t *testing.T) int {
	t.Helper()
	f, err := os.Open(os.DevNull)
	if err != nil {
		t.Fatalf("Cannot open %s: %v", os.DevNull, err)
	}
	fd := int(f.Fd())
	f.Close()
	return fd
}

// TestUploadToLeaksFileDescriptors demonstrates that GCS.UploadTo opens a local
// file reader via os.Open but never closes it, leaking one file descriptor per
// call. When storage.NewClient fails (e.g., invalid credentials), the function
// returns an error without closing the reader opened at the top of the function.
func TestUploadToLeaksFileDescriptors(t *testing.T) {
	// Force storage.NewClient to fail by pointing to a nonexistent credentials file.
	origCreds := os.Getenv("GOOGLE_APPLICATION_CREDENTIALS")
	os.Setenv("GOOGLE_APPLICATION_CREDENTIALS", "/nonexistent/invalid-credentials.json")
	defer os.Setenv("GOOGLE_APPLICATION_CREDENTIALS", origCreds)

	dir := t.TempDir()
	numFiles := 20

	// Create temp files for UploadTo to open.
	files := make([]string, numFiles)
	for i := 0; i < numFiles; i++ {
		f := filepath.Join(dir, fmt.Sprintf("upload-leak-test-%d.txt", i))
		if err := os.WriteFile(f, []byte("test payload data"), 0644); err != nil {
			t.Fatalf("Failed to create temp file: %v", err)
		}
		files[i] = f
	}

	// Stabilize FD count by forcing a GC + allowing finalizers to run.
	runtime.GC()

	beforeFD := nextAvailableFD(t)

	g := &GCS{}
	for _, f := range files {
		err := g.UploadTo("", "fake-bucket", f)
		// We expect UploadTo to fail (bad credentials), but the reader leak still occurs.
		if err == nil {
			t.Fatalf("Expected UploadTo to fail with invalid credentials, but it succeeded")
		}
	}

	afterFD := nextAvailableFD(t)
	leaked := afterFD - beforeFD

	t.Logf("Next available FD before: %d, after: %d, leaked: %d (expected >= %d)", beforeFD, afterFD, leaked, numFiles)

	// The bug: each UploadTo call opens a reader via os.Open but never closes it
	// when storage.NewClient fails. The next available FD number should advance
	// by at least numFiles because each leaked reader occupies one FD slot.
	if leaked < numFiles {
		t.Errorf("Expected at least %d leaked file descriptors, but only %d were leaked (before FD: %d, after FD: %d). "+
			"The reader may have been closed — hypothesis not confirmed.",
			numFiles, leaked, beforeFD, afterFD)
	}
}
```

### Test Output

```
=== RUN   TestUploadToLeaksFileDescriptors
    upload_to_gcs_test.go:65: Next available FD before: 4, after: 24, leaked: 20 (expected >= 20)
--- PASS: TestUploadToLeaksFileDescriptors (0.01s)
PASS
ok  	github.com/stellar/stellar-etl/v2/cmd	1.827s
```
