# 002: Parquet upload happens before regeneration in four export commands

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial cloud parquet export contains stale prior-run data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` call `MaybeUpload(parquetPath)` before `WriteParquet(...)`. Because `(*GCS).UploadTo` opens and streams the file currently on disk, the first run fails before creating the parquet artifact, while any reused parquet path uploads stale prior-run bytes to cloud storage and only then regenerates the local parquet file. Downstream consumers reading the uploaded parquet object receive silently stale data.

## Root Cause

The four affected commands reverse the otherwise-correct parquet lifecycle: upload first, write second. `MaybeUpload` delegates to `(*GCS).UploadTo`, which immediately calls `os.Open(path)` on the current parquet path, copies those bytes to cloud storage, verifies the object, and deletes the local file. If the command has not yet generated the new parquet file, the uploader either errors on a missing path or uploads whatever stale bytes were left from a previous run.

## Reproduction

Run any affected export command with `--write-parquet` and cloud upload enabled.

1. On a fresh parquet path, the command reaches the parquet block, tries to upload a file that does not exist yet, and fatals before writing the parquet artifact.
2. On a reused parquet path, the command uploads the old on-disk parquet object, deletes that local file, and only afterward rewrites the local parquet path with current data.
3. The cloud parquet object therefore reflects stale prior-run output while the local parquet file reflects the current run.

## Affected Code

- `cmd/export_ledgers.go:ledgersCmd.Run:68-72` — uploads `parquetPath` before generating the new ledger parquet.
- `cmd/export_transactions.go:transactionsCmd.Run:61-65` — uploads `parquetPath` before generating the new transaction parquet.
- `cmd/export_operations.go:operationsCmd.Run:61-65` — uploads `parquetPath` before generating the new operation parquet.
- `cmd/export_trades.go:tradesCmd.Run:66-70` — uploads `parquetPath` before generating the new trade parquet.
- `cmd/command_utils.go:MaybeUpload:123-146` — turns the uploader error into a fatal exit when the parquet path is missing.
- `cmd/upload_to_gcs.go:(*GCS).UploadTo:25-73` — opens the current file contents, uploads them, verifies the object, and deletes the local path.

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestParquetUploadCalledBeforeWrite` and `TestParquetUploadReadsStaleFile`
- **Test language**: Go
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./cmd/... -run 'TestParquetUpload' -v`
  4. Observe that the affected commands order upload before write, and that the uploader sends stale parquet bytes already on disk.

### Test Body

```go
package cmd

import (
	"bytes"
	"encoding/json"
	"go/ast"
	"go/parser"
	"go/token"
	"io"
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"strconv"
	"strings"
	"sync"
	"testing"
)

func TestParquetUploadCalledBeforeWrite(t *testing.T) {
	buggyFiles := []string{
		"cmd/export_ledgers.go",
		"cmd/export_transactions.go",
		"cmd/export_operations.go",
		"cmd/export_trades.go",
	}
	correctFiles := []string{
		"cmd/export_effects.go",
		"cmd/export_assets.go",
		"cmd/export_contract_events.go",
	}

	for _, filename := range buggyFiles {
		t.Run(filename, func(t *testing.T) {
			order := parquetBlockCallOrder(t, filename)
			if order != "MaybeUpload-before-WriteParquet" {
				t.Fatalf("expected buggy ordering in %s, got %s", filename, order)
			}
		})
	}

	for _, filename := range correctFiles {
		t.Run(filename, func(t *testing.T) {
			order := parquetBlockCallOrder(t, filename)
			if order != "WriteParquet-before-MaybeUpload" {
				t.Fatalf("expected correct ordering in %s, got %s", filename, order)
			}
		})
	}

	t.Run("missing parquet path fails before write", func(t *testing.T) {
		tmpDir := t.TempDir()
		missingPath := filepath.Join(tmpDir, "missing.parquet")
		err := newGCS("", "test-bucket").UploadTo("", "test-bucket", missingPath)
		if err == nil {
			t.Fatal("expected upload to fail for a missing parquet file")
		}
		if !strings.Contains(err.Error(), "failed to open file") {
			t.Fatalf("expected open-file error, got %v", err)
		}
	})
}

func TestParquetUploadReadsStaleFile(t *testing.T) {
	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "output.parquet")
	staleContent := []byte("stale parquet from previous run")
	freshContent := []byte("fresh parquet from current run")

	if err := os.WriteFile(parquetPath, staleContent, 0o644); err != nil {
		t.Fatal(err)
	}

	server := newFakeGCSServer(t)
	defer server.Close()

	t.Setenv("STORAGE_EMULATOR_HOST", server.URL())

	if err := newGCS("", "test-bucket").UploadTo("", "test-bucket", parquetPath); err != nil {
		t.Fatalf("unexpected upload error: %v", err)
	}

	uploaded := server.UploadedBody()
	if !bytes.Contains(uploaded, staleContent) {
		t.Fatalf("expected uploaded payload to contain stale content %q, got %q", staleContent, uploaded)
	}
	if bytes.Contains(uploaded, freshContent) {
		t.Fatalf("uploaded payload unexpectedly contained fresh content %q", freshContent)
	}

	if _, err := os.Stat(parquetPath); !os.IsNotExist(err) {
		t.Fatalf("expected upload to delete the local parquet file, stat err=%v", err)
	}

	if err := os.WriteFile(parquetPath, freshContent, 0o644); err != nil {
		t.Fatal(err)
	}

	rewritten, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatal(err)
	}
	if !bytes.Equal(rewritten, freshContent) {
		t.Fatalf("expected local file to be rewritten with fresh content, got %q", rewritten)
	}
	if bytes.Equal(uploaded, rewritten) {
		t.Fatal("expected uploaded bytes to differ from the freshly written local parquet file")
	}
}

func parquetBlockCallOrder(t *testing.T, filename string) string {
	t.Helper()

	fset := token.NewFileSet()
	node, err := parser.ParseFile(fset, filename, nil, 0)
	if err != nil {
		t.Fatalf("failed to parse %s: %v", filename, err)
	}

	var order []string
	ast.Inspect(node, func(n ast.Node) bool {
		ifStmt, ok := n.(*ast.IfStmt)
		if !ok {
			return true
		}

		selector, ok := ifStmt.Cond.(*ast.SelectorExpr)
		if !ok || selector.Sel.Name != "WriteParquet" {
			return true
		}
		ident, ok := selector.X.(*ast.Ident)
		if !ok || ident.Name != "commonArgs" {
			return true
		}

		for _, stmt := range ifStmt.Body.List {
			exprStmt, ok := stmt.(*ast.ExprStmt)
			if !ok {
				continue
			}
			call, ok := exprStmt.X.(*ast.CallExpr)
			if !ok {
				continue
			}
			ident, ok := call.Fun.(*ast.Ident)
			if !ok {
				continue
			}
			switch ident.Name {
			case "MaybeUpload", "WriteParquet":
				order = append(order, ident.Name)
			}
		}

		return true
	})

	if len(order) < 2 {
		t.Fatalf("expected both MaybeUpload and WriteParquet in %s, got %v", filename, order)
	}

	if order[0] == "MaybeUpload" && order[1] == "WriteParquet" {
		return "MaybeUpload-before-WriteParquet"
	}
	if order[0] == "WriteParquet" && order[1] == "MaybeUpload" {
		return "WriteParquet-before-MaybeUpload"
	}

	t.Fatalf("unexpected call order in %s: %v", filename, order)
	return ""
}

type fakeGCSServer struct {
	server     *httptest.Server
	mu         sync.Mutex
	uploaded   []byte
	uploadPath string
}

func newFakeGCSServer(t *testing.T) *fakeGCSServer {
	t.Helper()

	state := &fakeGCSServer{}
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch {
		case strings.HasPrefix(r.URL.Path, "/upload/storage/v1/b/"):
			body, err := io.ReadAll(r.Body)
			if err != nil {
				t.Fatalf("failed to read upload body: %v", err)
			}
			state.mu.Lock()
			state.uploaded = append([]byte(nil), body...)
			state.uploadPath = r.URL.Query().Get("name")
			state.mu.Unlock()
			writeObjectResponse(w, "test-bucket", state.uploadPath, len(body))
		case strings.HasPrefix(r.URL.Path, "/storage/v1/b/"):
			state.mu.Lock()
			name := state.uploadPath
			size := len(state.uploaded)
			state.mu.Unlock()
			writeObjectResponse(w, "test-bucket", name, size)
		default:
			t.Fatalf("unexpected GCS emulator request: %s %s", r.Method, r.URL.String())
		}
	})

	state.server = httptest.NewServer(handler)
	return state
}

func (s *fakeGCSServer) URL() string {
	return s.server.URL
}

func (s *fakeGCSServer) Close() {
	s.server.Close()
}

func (s *fakeGCSServer) UploadedBody() []byte {
	s.mu.Lock()
	defer s.mu.Unlock()
	return append([]byte(nil), s.uploaded...)
}

func writeObjectResponse(w http.ResponseWriter, bucket, name string, size int) {
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(map[string]string{
		"bucket": bucket,
		"name":   name,
		"size":   strconv.Itoa(size),
	})
}
```

## Expected vs Actual Behavior

- **Expected**: The exporter should generate the current parquet artifact first and only then upload that freshly written file to cloud storage.
- **Actual**: The four affected commands upload whatever is already at `parquetPath` before regenerating the parquet file, which either fatals on a missing file or silently uploads stale prior-run parquet bytes.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC proves the four export commands reverse the upload/write order and separately proves that the production GCS uploader reads the bytes currently on disk.
2. Realistic preconditions: YES — operators commonly reuse deterministic output paths while enabling `--write-parquet` plus cloud upload for recurring exports.
3. Bug vs by-design: BUG — sibling commands (`export_effects`, `export_assets`, `export_contract_events`) already use the correct write-then-upload order.
4. Final severity: High — the uploaded parquet object can contain structurally wrong, stale ledger data while looking valid to downstream readers.
5. In scope: YES — this is a concrete data-correctness issue in the repository’s export pipeline, not an upstream SDK problem.
6. Test correctness: CORRECT — the PoC does not rely on mocks for the production uploader logic, and the stale-data assertion checks the bytes actually sent to the emulator-backed upload endpoint.
7. Alternative explanations: NONE
8. Novelty: APPEARS NOVEL — cross-finding duplicate handling is orchestrator-owned.

## Suggested Fix

Swap the parquet calls in `export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` so `WriteParquet(...)` always runs before `MaybeUpload(..., parquetPath)`. Consider extracting the shared JSON/parquet finalization flow into a helper so the command template cannot drift again.
