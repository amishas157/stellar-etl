# 008: `--extra-fields` overwrites canonical JSON export columns

**Date**: 2026-04-11
**Severity**: High
**Impact**: structural data corruption
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

`ExportEntry` decodes each transformed row into a generic JSON map and then blindly copies every `--extra-fields` entry into that same map. When an operator supplies a key that collides with a canonical export column such as `ledger_sequence`, `fee_charged`, `id`, or `successful`, the helper silently replaces the real ledger-derived value and re-serializes the row with the injected string value instead.

Because the overwrite happens only in the JSON path, the same export run can emit conflicting JSON and Parquet rows for the same record. The README and flag help both describe `--extra-fields` as appended metadata, so this mutation of canonical columns is not documented behavior.

## Root Cause

`cmd.ExportEntry` marshals the typed row, decodes it into `map[string]interface{}`, then unconditionally assigns `i[k] = v` for every extra field before re-marshaling. There is no collision check, no reserved-key validation during flag parsing, and no namespacing of metadata fields, so user-supplied metadata can replace any canonical JSON column.

## Reproduction

Run any JSON-exporting command with a colliding metadata key, for example `--extra-fields ledger_sequence=0,fee_charged=9999999`. During normal execution the export command passes that map to `ExportEntry`, which overwrites the decoded row map and writes the injected string values into the JSON output while the typed structs used for Parquet remain unchanged.

## Affected Code

- `cmd/command_utils.go:ExportEntry:55-86` — decodes the canonical row and blindly overwrites keys from `extra`
- `internal/utils/main.go:AddCommonFlags:231-246` — exposes arbitrary `--extra-fields` keys while documenting them as appended metadata
- `cmd/export_transactions.go:Run:17-66` — representative exporter that forwards `commonArgs.Extra` into `ExportEntry` (the same pattern is used across JSON exporters)

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestExtraFieldsOverwriteCanonicalColumns`
- **Test language**: go
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package cmd

import (
	"bytes"
	"encoding/json"
	"os"
	"path/filepath"
	"testing"

	"github.com/stellar/stellar-etl/v2/internal/transform"
)

func TestExtraFieldsOverwriteCanonicalColumns(t *testing.T) {
	entry := transform.TransactionOutput{
		TransactionHash: "txhash",
		LedgerSequence:  42,
		FeeCharged:      12345,
		Successful:      true,
		TransactionID:   99,
	}

	extra := map[string]string{
		"ledger_sequence": "0",
		"fee_charged":     "9999999",
		"id":              "injected-id",
		"successful":      "false",
		"review_tag":      "poc",
	}

	outPath := filepath.Join(t.TempDir(), "output.json")
	outFile, err := os.Create(outPath)
	if err != nil {
		t.Fatalf("create temp file: %v", err)
	}
	defer outFile.Close()

	if _, err := ExportEntry(entry, outFile, extra); err != nil {
		t.Fatalf("ExportEntry returned error: %v", err)
	}

	if _, err := outFile.Seek(0, 0); err != nil {
		t.Fatalf("seek output: %v", err)
	}

	decoder := json.NewDecoder(outFile)
	decoder.UseNumber()

	var result map[string]interface{}
	if err := decoder.Decode(&result); err != nil {
		t.Fatalf("decode output: %v", err)
	}

	if got := result["transaction_hash"]; got != "txhash" {
		t.Fatalf("expected transaction_hash to remain canonical, got %#v", got)
	}

	for key, want := range map[string]string{
		"ledger_sequence": "0",
		"fee_charged":     "9999999",
		"id":              "injected-id",
		"successful":      "false",
	} {
		got, ok := result[key].(string)
		if !ok {
			t.Fatalf("expected %s to be overwritten as string, got %T (%v)", key, result[key], result[key])
		}
		if got != want {
			t.Fatalf("expected %s=%q, got %q", key, want, got)
		}
	}

	if got := result["review_tag"]; got != "poc" {
		t.Fatalf("expected non-colliding metadata to be appended, got %#v", got)
	}

	raw, err := os.ReadFile(outPath)
	if err != nil {
		t.Fatalf("read output: %v", err)
	}
	if !bytes.Contains(raw, []byte(`"fee_charged":"9999999"`)) {
		t.Fatalf("expected overwritten JSON string in output, got %s", raw)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `--extra-fields` should append metadata without mutating any canonical export column, or the command should reject colliding keys.
- **Actual**: colliding keys overwrite canonical JSON columns and convert typed values such as numbers and booleans into injected strings.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production `ExportEntry` helper with a real `transform.TransactionOutput` and colliding `extra` keys, then decodes the emitted JSON row.
2. Realistic preconditions: YES — `--extra-fields` is a documented public CLI flag and accepts arbitrary `key=value` pairs during normal exports.
3. Bug vs by-design: BUG — both the flag help text and README say the fields are appended metadata; silently replacing canonical ledger-derived columns contradicts that contract.
4. Final severity: High — the bug silently corrupts arbitrary canonical JSON columns and can desynchronize JSON from Parquet output, but it requires an explicit operator-supplied colliding flag rather than arising from ordinary ledger contents.
5. In scope: YES — this is a concrete JSON export path that produces wrong persisted output without crashing.
6. Test correctness: CORRECT — the test uses the production code path, confirms a non-colliding field is appended, and shows colliding canonical fields change value and type.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Reject `--extra-fields` keys that collide with canonical JSON columns, or only merge metadata when the key is absent. If arbitrary metadata must remain flexible, namespace it under a dedicated object instead of flattening it into the top-level export row.
