# 003: Stale Parquet uploaded before regeneration

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_ledgers`, `export_transactions`, `export_operations`, and `export_trades` upload the Parquet path to GCS before regenerating that file for the current run. If the path already contains a previous export, the cloud object is silently overwritten with stale but valid Parquet rows while the local file ends up holding the new ledger range.

## Root Cause

These four commands call `MaybeUpload(..., parquetPath)` before `WriteParquet(...)`. `MaybeUpload()` delegates to `GCS.UploadTo()`, which opens whatever file already exists at `parquetPath`, copies those bytes to the bucket, verifies the object exists, and then deletes the local file. Only after that does `WriteParquet()` create the new Parquet file for the current export.

## Reproduction

Any normal rerun of an affected command with `--write-parquet --cloud-provider gcp` and a reused `--parquet-output` path hits this behavior. That precondition is realistic because the CLI defaults `--parquet-output` to fixed filenames like `exported_trades.parquet`, so repeated exports naturally reuse the same path unless callers override it.

## Affected Code

- `cmd/export_ledgers.go:ledgersCmd.Run:70-72` - uploads `parquetPath` before regenerating it
- `cmd/export_transactions.go:transactionsCmd.Run:63-65` - uploads `parquetPath` before regenerating it
- `cmd/export_operations.go:operationsCmd.Run:63-65` - uploads `parquetPath` before regenerating it
- `cmd/export_trades.go:tradesCmd.Run:68-70` - uploads `parquetPath` before regenerating it
- `cmd/command_utils.go:MaybeUpload:123-146` - immediately dispatches the upload when cloud output is enabled
- `cmd/upload_to_gcs.go:GCS.UploadTo:25-71` - opens the current local file, uploads it, verifies the object, and deletes the file afterward

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestStaleParquetUploadBeforeRegeneration`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Create `cmd/data_integrity_poc_test.go` with the test body below.
  3. Run `go test ./cmd/... -run TestStaleParquetUploadBeforeRegeneration -v`
  4. Observe that the uploaded object bytes match the stale prior-run Parquet file, while the regenerated local file contains the fresh row.

### Test Body

```go
package cmd

import (
	"bytes"
	"fmt"
	"io"
	"mime"
	"mime/multipart"
	"net/http"
	"net/http/httptest"
	"net/url"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/writer"
)

func writeParquetForTest(t *testing.T, data []transform.SchemaParquet, path string, schema interface{}) {
	t.Helper()

	parquetFile, err := local.NewLocalFileWriter(path)
	if err != nil {
		t.Fatalf("could not create parquet file: %v", err)
	}
	defer parquetFile.Close()

	pw, err := writer.NewParquetWriter(parquetFile, schema, 1)
	if err != nil {
		t.Fatalf("could not create parquet writer: %v", err)
	}
	defer pw.WriteStop()

	for _, record := range data {
		if err := pw.Write(record.ToParquet()); err != nil {
			t.Fatalf("could not write parquet record: %v", err)
		}
	}
}

func newFakeGCSServer(t *testing.T) (*httptest.Server, map[string][]byte) {
	t.Helper()

	uploads := map[string][]byte{}
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch {
		case r.Method == http.MethodPost &&
			strings.HasPrefix(r.URL.Path, "/upload/storage/v1/b/") &&
			strings.HasSuffix(r.URL.Path, "/o"):
			contentType := r.Header.Get("Content-Type")
			mediaType, params, err := mime.ParseMediaType(contentType)
			if err != nil {
				t.Fatalf("could not parse content type %q: %v", contentType, err)
			}
			if !strings.HasPrefix(mediaType, "multipart/") {
				t.Fatalf("expected multipart upload, got %q", mediaType)
			}

			mr := multipart.NewReader(r.Body, params["boundary"])
			var media []byte
			for {
				part, err := mr.NextPart()
				if err == io.EOF {
					break
				}
				if err != nil {
					t.Fatalf("could not read multipart upload: %v", err)
				}
				body, err := io.ReadAll(part)
				if err != nil {
					t.Fatalf("could not read multipart part: %v", err)
				}
				media = body
			}

			objectName := r.URL.Query().Get("name")
			if objectName == "" {
				t.Fatal("missing object name in upload request")
			}
			uploads[objectName] = media

			fmt.Fprintf(w, `{"bucket":%q,"name":%q,"size":"%d"}`, "test-bucket", objectName, len(media))

		case r.Method == http.MethodGet &&
			strings.HasPrefix(r.URL.Path, "/storage/v1/b/") &&
			strings.Contains(r.URL.Path, "/o/"):
			objectPath := strings.SplitN(r.URL.Path, "/o/", 2)[1]
			objectName, err := url.PathUnescape(objectPath)
			if err != nil {
				t.Fatalf("could not decode object path %q: %v", objectPath, err)
			}
			content, ok := uploads[objectName]
			if !ok {
				http.NotFound(w, r)
				return
			}

			fmt.Fprintf(w, `{"bucket":%q,"name":%q,"size":"%d"}`, "test-bucket", objectName, len(content))

		default:
			http.NotFound(w, r)
		}
	}))

	return server, uploads
}

func TestStaleParquetUploadBeforeRegeneration(t *testing.T) {
	tmpDir := t.TempDir()
	parquetPath := filepath.Join(tmpDir, "trades.parquet")

	staleTrades := []transform.SchemaParquet{
		transform.TradeOutput{HistoryOperationID: 111, SellingAccountAddress: "STALE_SELLER"},
	}
	writeParquetForTest(t, staleTrades, parquetPath, new(transform.TradeOutputParquet))

	staleContent, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("failed to read stale parquet file: %v", err)
	}

	server, uploads := newFakeGCSServer(t)
	defer server.Close()
	t.Setenv("STORAGE_EMULATOR_HOST", server.URL)

	cloudStorage := newGCS("", "test-bucket")
	if err := cloudStorage.UploadTo("", "test-bucket", parquetPath); err != nil {
		t.Fatalf("UploadTo failed: %v", err)
	}

	if _, err := os.Stat(parquetPath); !os.IsNotExist(err) {
		t.Fatalf("expected UploadTo to delete local parquet file, got stat err=%v", err)
	}

	freshTrades := []transform.SchemaParquet{
		transform.TradeOutput{HistoryOperationID: 999, SellingAccountAddress: "FRESH_SELLER"},
	}
	writeParquetForTest(t, freshTrades, parquetPath, new(transform.TradeOutputParquet))

	freshContent, err := os.ReadFile(parquetPath)
	if err != nil {
		t.Fatalf("failed to read fresh parquet file: %v", err)
	}

	uploadedContent, ok := uploads[parquetPath]
	if !ok {
		t.Fatalf("fake GCS never received upload for %s", parquetPath)
	}

	if !bytes.Equal(uploadedContent, staleContent) {
		t.Fatal("expected uploaded content to match the stale parquet file")
	}

	if bytes.Equal(uploadedContent, freshContent) {
		t.Fatal("uploaded content unexpectedly matches the fresh parquet file")
	}
}
```

## Expected vs Actual Behavior

- **Expected**: The uploaded GCS object should contain the Parquet rows generated for the current export run.
- **Actual**: The uploaded object contains the prior run's Parquet bytes, while the local path is regenerated afterward with the current run's rows.

## Adversarial Review

1. Exercises claimed bug: YES - the PoC writes a real stale Parquet file, runs the real `GCS.UploadTo()` implementation against a local storage emulator, then regenerates the same path with fresh data.
2. Realistic preconditions: YES - `--write-parquet` and `--cloud-provider gcp` are normal CLI flags, and the default `--parquet-output` values are fixed filenames that encourage path reuse across runs.
3. Bug vs by-design: BUG - sibling commands such as `export_effects`, `export_assets`, `export_contract_events`, and `export_ledger_entry_changes` use the correct write-then-upload order.
4. Final severity: High - this silently publishes structurally wrong export data to cloud storage while the command otherwise completes successfully.
5. In scope: YES - it is a concrete production export path producing wrong output consumed outside the process.
6. Test correctness: CORRECT - the revised PoC exercises the actual upload helper rather than a hand-simulated file read, so the observed stale upload is attributable to the real production code path.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Write the Parquet file before calling `MaybeUpload()` in the four affected commands, and ideally stage Parquet output via a fresh temporary file or otherwise guarantee that uploads only read artifacts produced by the current run.
