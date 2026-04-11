# 007: Zero-limit ledger readers still export the first ledger

**Date**: 2026-04-11
**Severity**: High
**Impact**: Structural data corruption: `--limit 0` silently emits ledger and token-transfer rows
**Subsystem**: data-input
**Final review by**: gpt-5.4, high

## Summary

`AddArchiveFlags()` documents `--limit` as the maximum number of exported objects, so `--limit 0` should produce an empty result set. Instead, both `GetLedgers()` and `GetLedgersHistoryArchive()` append the current ledger before enforcing the non-negative limit, so a zero-limit export still returns the first ledger in-range.

This is visible in normal operation. On pubnet, `export_ledgers -s 30822015 -e 30822015 -l 0` writes a non-empty ledger export on both reader paths, and `export_token_transfer -s 30822015 -e 30822015 -l 0` writes `57` token-transfer rows because that command reuses `GetLedgers()`.

## Root Cause

`internal/input/ledgers.go` and `internal/input/ledgers_history_archive.go` both use an append-then-check pattern:

```go
ledgerSlice = append(ledgerSlice, ledgerLCM)
if int64(len(ledgerSlice)) >= limit && limit >= 0 {
	break
}
```

When `limit == 0`, the first append makes `len(ledgerSlice) == 1`, and only then does the guard fire. Sibling readers such as `GetTransactions()` instead guard entry with `len(slice) < limit || limit < 0`, so they correctly return zero rows for a zero limit.

## Reproduction

During normal operation, the issue reproduces on real pubnet data:

```bash
./stellar-etl export_ledgers -s 30822015 -e 30822015 -l 0 -o /tmp/ledger-limit0.json
./stellar-etl export_ledgers -s 30822015 -e 30822015 -l 0 --captive-core -o /tmp/ledger-limit0-archive.json
./stellar-etl export_token_transfer -s 30822015 -e 30822015 -l 0 -o /tmp/token-limit0.json

wc -c /tmp/ledger-limit0.json /tmp/ledger-limit0-archive.json
wc -l /tmp/token-limit0.json
```

Observed results during final review:

- both `export_ledgers` commands exited successfully and wrote `1181` bytes instead of an empty file
- `export_token_transfer` exited successfully and wrote `57` rows instead of `0`

## Affected Code

- `internal/input/ledgers.go:14-90` — appends a ledger before enforcing the limit
- `internal/input/ledgers_history_archive.go:10-34` — repeats the same append-then-check limit logic
- `internal/input/transactions.go:23-70` — contrasting sibling reader that correctly honors `limit == 0`
- `cmd/export_ledgers.go:21-32` — exposes both buggy reader paths through the CLI
- `cmd/export_token_transfers.go:21-29` — reuses `GetLedgers()` and therefore leaks first-ledger token-transfer rows
- `internal/utils/main.go:248-254` — documents `--limit` as the maximum number of exported objects

## PoC

- **Target test file**: `internal/input/data_integrity_poc_test.go`
- **Test name**: `TestZeroLimitLedgerReadersExportFirstLedger`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
)

func TestZeroLimitLedgerReadersExportFirstLedger(t *testing.T) {
	env := utils.GetEnvironmentDetails(utils.CommonFlagValues{
		DatastorePath: "sdf-ledger-close-meta/v1/ledgers",
		BufferSize:    200,
		NumWorkers:    10,
		RetryLimit:    3,
		RetryWait:     5,
	})

	const (
		start = uint32(30822015)
		end   = uint32(30822015)
		limit = int64(0)
	)

	ledgers, err := GetLedgers(start, end, limit, env, false)
	if err != nil {
		t.Fatalf("GetLedgers returned error: %v", err)
	}
	if len(ledgers) != 1 {
		t.Fatalf("expected GetLedgers to export exactly 1 ledger with limit=0, got %d", len(ledgers))
	}
	if got := ledgers[0].Ledger.Header.Header.LedgerSeq; got != xdr.Uint32(start) {
		t.Fatalf("expected GetLedgers to export ledger %d, got %d", start, got)
	}

	archiveLedgers, err := GetLedgersHistoryArchive(start, end, limit, env, true)
	if err != nil {
		t.Fatalf("GetLedgersHistoryArchive returned error: %v", err)
	}
	if len(archiveLedgers) != 1 {
		t.Fatalf("expected GetLedgersHistoryArchive to export exactly 1 ledger with limit=0, got %d", len(archiveLedgers))
	}
	if got := archiveLedgers[0].Ledger.Header.Header.LedgerSeq; got != xdr.Uint32(start) {
		t.Fatalf("expected GetLedgersHistoryArchive to export ledger %d, got %d", start, got)
	}

	txs, err := GetTransactions(start, end, limit, env, false)
	if err != nil {
		t.Fatalf("GetTransactions returned error: %v", err)
	}
	if len(txs) != 0 {
		t.Fatalf("expected GetTransactions to honor limit=0 and return 0 rows, got %d", len(txs))
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `--limit 0` should stop all three readers before they append anything, so `export_ledgers` and `export_token_transfer` should emit no rows at all.
- **Actual**: `GetLedgers()` and `GetLedgersHistoryArchive()` both retain the first ledger before checking the limit, so zero-limit exports still emit one ledger and all token-transfer rows from that first ledger.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC calls the production readers directly on real pubnet ledger `30822015`, and independent CLI runs showed non-empty outputs for `export_ledgers -l 0` and `57` token-transfer rows for `export_token_transfer -l 0`.
2. Realistic preconditions: YES — `--limit 0` is accepted by the shipping CLI, and the affected readers are the normal production paths for ledger and token-transfer exports.
3. Bug vs by-design: BUG — `AddArchiveFlags()` defines `--limit` as the maximum number of exported objects, and sibling readers already implement the expected zero-limit behavior.
4. Final severity: High — the exporter returns structurally wrong non-financial output without error, including a supposedly empty ledger export and a zero-limit token-transfer export that still writes `57` rows.
5. In scope: YES — this is a concrete production code path that silently produces wrong output.
6. Test correctness: CORRECT — the final PoC uses the real reader functions and real ledger data rather than replaying loop logic in isolation.
7. Alternative explanations: NONE — once the append happens before the non-negative limit check, a zero limit necessarily retains the first ledger.
8. Novelty: NOVEL

## Suggested Fix

Short-circuit both ledger readers before the first append when `limit == 0`, or rewrite them to use the same pre-append guard pattern already used by `GetTransactions()`, `GetOperations()`, and `GetTrades()`. Add regression tests for `GetLedgers()`, `GetLedgersHistoryArchive()`, `export_ledgers`, and `export_token_transfer` with `--limit 0`.
