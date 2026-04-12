# 014: JSON exporters ignore close failures

**Date**: 2026-04-12
**Severity**: Medium
**Impact**: Data loss or silent failure under specific conditions
**Subsystem**: cli-commands
**Final review by**: gpt-5.4, high

## Summary

All nine one-shot JSON export commands call `outFile.Close()` as a bare statement and discard its error. If the final file close surfaces a delayed writeback failure, the command still logs success stats and can continue into `MaybeUpload(...)`, so operators get a success-shaped completion after a failed file finalization.

## Root Cause

Each command writes JSON rows to a writable `*os.File`, then invokes `outFile.Close()` without checking the returned error. `close(2)` can report `EIO` for previously uncommitted writes, so the command layer is currently discarding a real part of the writable-file error surface.

## Reproduction

Append the PoC below to `cmd/data_integrity_poc_test.go`, run the targeted test, and inspect the real exporter sources. The dynamic test forces a non-nil `Close()` result on the same `*os.File` type used by production, while the AST test proves every affected command still ignores that result and reaches `MaybeUpload(...)` afterward.

## Affected Code

- `cmd/export_ledgers.go:ledgersCmd.Run:63-68` — ignores `outFile.Close()` and continues to stats/upload
- `cmd/export_transactions.go:transactionsCmd.Run:56-61` — same unchecked close pattern
- `cmd/export_operations.go:operationsCmd.Run:56-61` — same unchecked close pattern
- `cmd/export_effects.go:effectsCmd.Run:59-64` — same unchecked close pattern
- `cmd/export_trades.go:tradesCmd.Run:61-66` — same unchecked close pattern
- `cmd/export_assets.go:assetsCmd.Run:72-77` — same unchecked close pattern
- `cmd/export_contract_events.go:contractEventsCmd.Run:57-61` — same unchecked close pattern
- `cmd/export_token_transfers.go:tokenTransfersCmd.Run:57-62` — same unchecked close pattern
- `cmd/export_ledger_transaction.go:ledgerTransactionsCmd.Run:51-56` — same unchecked close pattern

## PoC

- **Target test file**: `cmd/data_integrity_poc_test.go`
- **Test name**: `TestAllExportCommandsDiscardCloseError` and `TestProductionWritePathThenCloseErrorDiscarded`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run `go test ./cmd/... -run 'TestAllExportCommandsDiscardCloseError|TestProductionWritePathThenCloseErrorDiscarded' -v`.

### Test Body

```go
package cmd

import (
	"go/ast"
	"go/parser"
	"go/token"
	"os"
	"path/filepath"
	"syscall"
	"testing"
)

// TestAllExportCommandsDiscardCloseError proves the JSON export commands call
// outFile.Close() as a bare statement and only call MaybeUpload afterward.
func TestAllExportCommandsDiscardCloseError(t *testing.T) {
	exportFiles := []string{
		"cmd/export_ledgers.go",
		"cmd/export_transactions.go",
		"cmd/export_operations.go",
		"cmd/export_effects.go",
		"cmd/export_assets.go",
		"cmd/export_trades.go",
		"cmd/export_contract_events.go",
		"cmd/export_token_transfers.go",
		"cmd/export_ledger_transaction.go",
	}

	for _, file := range exportFiles {
		t.Run(filepath.Base(file), func(t *testing.T) {
			fset := token.NewFileSet()
			node, err := parser.ParseFile(fset, file, nil, 0)
			if err != nil {
				t.Fatalf("parse %s: %v", file, err)
			}

			var bareClosePos token.Pos
			var maybeUploadPos token.Pos
			checkedCloseFound := false

			ast.Inspect(node, func(n ast.Node) bool {
				switch n := n.(type) {
				case *ast.ExprStmt:
					if isOutFileCloseCall(n.X) && bareClosePos == token.NoPos {
						bareClosePos = n.Pos()
					}
					if isMaybeUploadCall(n.X) && maybeUploadPos == token.NoPos {
						maybeUploadPos = n.Pos()
					}
				case *ast.AssignStmt:
					for _, rhs := range n.Rhs {
						if isOutFileCloseCall(rhs) {
							checkedCloseFound = true
						}
					}
				case *ast.IfStmt:
					assign, ok := n.Init.(*ast.AssignStmt)
					if !ok {
						return true
					}
					for _, rhs := range assign.Rhs {
						if isOutFileCloseCall(rhs) {
							checkedCloseFound = true
						}
					}
				}
				return true
			})

			if bareClosePos == token.NoPos {
				t.Fatalf("no bare outFile.Close() found in %s", file)
			}
			if maybeUploadPos == token.NoPos {
				t.Fatalf("no MaybeUpload call found in %s", file)
			}
			if checkedCloseFound {
				t.Fatalf("found checked outFile.Close() in %s", file)
			}
			if maybeUploadPos <= bareClosePos {
				t.Fatalf("MaybeUpload does not occur after bare outFile.Close() in %s", file)
			}
		})
	}
}

// TestProductionWritePathThenCloseErrorDiscarded shows the production helpers
// can surface a close error while the export-command pattern would continue
// into success logging and MaybeUpload without handling it.
func TestProductionWritePathThenCloseErrorDiscarded(t *testing.T) {
	tmpDir := t.TempDir()
	outPath := filepath.Join(tmpDir, "export_output.txt")

	outFile := MustOutFile(outPath)
	entry := map[string]interface{}{
		"ledger_sequence": 1234,
		"hash":            "abc123def456",
	}

	if _, err := ExportEntry(entry, outFile, nil); err != nil {
		t.Fatalf("ExportEntry failed: %v", err)
	}

	rawFd := int(outFile.Fd())
	if err := syscall.Close(rawFd); err != nil {
		t.Fatalf("syscall.Close(%d): %v", rawFd, err)
	}

	closeErr := outFile.Close()
	if closeErr == nil {
		t.Skip("platform did not surface a close error")
	}

	PrintTransformStats(1, 0)
	MaybeUpload("", "", "", outPath)

	info, err := os.Stat(outPath)
	if err != nil {
		t.Fatalf("stat output after close error: %v", err)
	}
	if info.Size() == 0 {
		t.Fatalf("expected non-empty output file")
	}
}

func isOutFileCloseCall(expr ast.Expr) bool {
	call, ok := expr.(*ast.CallExpr)
	if !ok {
		return false
	}
	sel, ok := call.Fun.(*ast.SelectorExpr)
	if !ok {
		return false
	}
	ident, ok := sel.X.(*ast.Ident)
	if !ok {
		return false
	}
	return ident.Name == "outFile" && sel.Sel.Name == "Close"
}

func isMaybeUploadCall(expr ast.Expr) bool {
	call, ok := expr.(*ast.CallExpr)
	if !ok {
		return false
	}
	ident, ok := call.Fun.(*ast.Ident)
	return ok && ident.Name == "MaybeUpload"
}
```

## Expected vs Actual Behavior

- **Expected**: If `outFile.Close()` fails, the export command should stop, report a non-zero failure, and skip success stats and upload.
- **Actual**: The commands discard the `Close()` error, report success, and keep executing the post-close path.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC forces a real non-nil `Close()` result on the same writable `*os.File` type used by production, and the AST pass verifies the actual exporters ignore that result and continue to `MaybeUpload(...)`.
2. Realistic preconditions: YES — the deterministic `EBADF` stand-in is not itself the production trigger, but Darwin `close(2)` documents `EIO` for previously uncommitted writes, so writable close failures are a real OS-level condition.
3. Bug vs by-design: BUG — no code or documentation indicates writable close failures are intentionally ignored, and doing so contradicts the exporter’s other file-output error handling.
4. Final severity: Medium — this is silent operational data loss masking: the exporter can claim success after JSON file finalization failed, but it does not directly remap financial fields.
5. In scope: YES — it is a concrete CLI export correctness flaw affecting supported output workflows.
6. Test correctness: CORRECT — the test does not assume the bug; it separately proves a non-nil close result on the relevant file type and checks the real command sources for the unchecked close-plus-continue path.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Replace every bare `outFile.Close()` in the one-shot JSON exporters with checked error handling, e.g. `if err := outFile.Close(); err != nil { cmdLogger.Fatal(...) }`, and only print success stats or call `MaybeUpload(...)` after a successful close.
