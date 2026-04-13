# 024: GCS upload leaks source file descriptors

**Date**: 2026-04-13
**Severity**: Medium
**Impact**: data loss or silent failure under specific conditions
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`GCS.UploadTo()` opens the local artifact with `os.Open(path)` and never closes that reader on either the success path or any error path. Repeated cloud uploads therefore leave deleted source files and their descriptors live until a later GC cycle, so long-running or batched exports can hit the OS file-descriptor limit and stop producing later artifacts.

## Root Cause

`UploadTo()` creates `reader` before it creates the storage client and then returns from every path without calling `reader.Close()`. Because the function deletes the pathname after a successful upload, the missing close is easy to miss in manual testing even though the process still retains the descriptor and underlying inode.

## Reproduction

Run repeated successful `UploadTo()` calls in one process against a local storage emulator. Each call uploads and deletes a different local file, yet the next available file descriptor still increases by one per upload because the source reader is never closed.

## Affected Code

- `cmd/upload_to_gcs.go:25-73` — `UploadTo()` opens the local source file and returns on every path without closing it.
- `cmd/command_utils.go:123-146` — `MaybeUpload()` dispatches every cloud-enabled export through `UploadTo()`.
- `cmd/export_ledger_entry_changes.go:368-372` — batched change exports can invoke `MaybeUpload()` twice per resource per batch.

## PoC

- **Target test file**: `cmd/upload_to_gcs_poc_test.go`
- **Test name**: `TestUploadToLeaksFileDescriptors`
- **Test language**: go
- **How to run**: Create the target test file with the body below, then run `go test ./cmd/... -run TestUploadToLeaksFileDescriptors -v`.

### Test Body

```go
package cmd

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"net/url"
	"os"
	"runtime/debug"
	"strings"
	"testing"
)

func nextFD(t *testing.T) int {
	t.Helper()

	f, err := os.Open("/dev/null")
	if err != nil {
		t.Fatalf("probe fd: %v", err)
	}
	fd := int(f.Fd())
	if err := f.Close(); err != nil {
		t.Fatalf("close fd probe: %v", err)
	}
	return fd
}

func newFakeGCSServer() *httptest.Server {
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("Connection", "close")

		switch {
		case r.Method == http.MethodPost &&
			strings.HasPrefix(r.URL.Path, "/upload/storage/v1/b/") &&
			strings.HasSuffix(r.URL.Path, "/o"):
			name := r.URL.Query().Get("name")
			fmt.Fprintf(w, `{"kind":"storage#object","bucket":"bucket","name":%q}`, name)
		case r.Method == http.MethodGet &&
			strings.HasPrefix(r.URL.Path, "/storage/v1/b/bucket/o/"):
			name := strings.TrimPrefix(r.URL.Path, "/storage/v1/b/bucket/o/")
			if decoded, err := url.PathUnescape(name); err == nil {
				name = decoded
			}
			fmt.Fprintf(w, `{"kind":"storage#object","bucket":"bucket","name":%q}`, name)
		default:
			http.Error(w, "unexpected request", http.StatusNotFound)
		}
	}))
	server.Config.SetKeepAlivesEnabled(false)
	return server
}

func TestUploadToLeaksFileDescriptors(t *testing.T) {
	oldGCPercent := debug.SetGCPercent(-1)
	defer debug.SetGCPercent(oldGCPercent)

	server := newFakeGCSServer()
	defer server.Close()

	t.Setenv("STORAGE_EMULATOR_HOST", server.URL)

	dir := t.TempDir()
	oldWD, err := os.Getwd()
	if err != nil {
		t.Fatalf("get working directory: %v", err)
	}
	if err := os.Chdir(dir); err != nil {
		t.Fatalf("chdir to temp dir: %v", err)
	}
	defer func() {
		if err := os.Chdir(oldWD); err != nil {
			t.Fatalf("restore working directory: %v", err)
		}
	}()

	const uploads = 25
	gcs := &GCS{}
	fdBefore := nextFD(t)

	for i := 0; i < uploads; i++ {
		name := fmt.Sprintf("upload-%02d.txt", i)
		if err := os.WriteFile(name, []byte("payload"), 0o644); err != nil {
			t.Fatalf("write upload source %q: %v", name, err)
		}

		if err := gcs.UploadTo("", "bucket", name); err != nil {
			t.Fatalf("upload %q: %v", name, err)
		}

		if _, err := os.Stat(name); !os.IsNotExist(err) {
			t.Fatalf("expected %q to be deleted after upload, got err=%v", name, err)
		}
	}

	server.CloseClientConnections()
	fdAfter := nextFD(t)
	leaked := fdAfter - fdBefore

	t.Logf("next fd before=%d after=%d leaked=%d uploads=%d", fdBefore, fdAfter, leaked, uploads)

	if leaked < uploads {
		t.Fatalf("expected at least %d leaked file descriptors after successful UploadTo calls, got %d", uploads, leaked)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Each upload closes the local source file once the copy and verification steps finish, so repeated uploads keep the process FD count bounded.
- **Actual**: Each successful upload leaves one more deleted local file descriptor open until GC runs.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls `GCS.UploadTo()` itself and drives its successful upload, verification, and local-delete path.
2. Realistic preconditions: YES — any cloud-enabled export that performs many uploads in one process can trigger this, especially batched or continuous commands.
3. Bug vs by-design: BUG — there is no documented ownership transfer or intentional reliance on GC for descriptor cleanup.
4. Final severity: Medium — this is an operational correctness issue that can stop later uploads or output file creation once the process reaches the OS FD limit.
5. In scope: YES — the result is silent/late data loss during export, which matches the stated Medium impact category.
6. Test correctness: CORRECT — the harness closes emulator client connections before measuring FD growth, so the observed delta maps to leaked source-file readers rather than idle HTTP sockets.
7. Alternative explanations: NONE — after eliminating emulator connection noise, 25 successful uploads produced exactly 25 additional live descriptors.
8. Novelty: DUPLICATE-SUSPECT — the same root cause already appears in the global index under `cli-commands/006`; duplicate handling is left to the orchestrator.

## Suggested Fix

Call `defer reader.Close()` immediately after the `os.Open(path)` error check succeeds so every return path releases the source descriptor deterministically.
