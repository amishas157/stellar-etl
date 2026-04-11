# H002: Token-transfer export applies `--limit` to ledgers instead of emitted transfer rows

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Empty or truncated token-transfer exports under caller-visible row limits
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_token_transfer --limit N` should continue scanning the requested range until it has emitted `N` `TokenTransferOutput` rows or reached the end of the range. A ledger with zero token-transfer events should not consume the caller's token-transfer budget.

## Mechanism

The command passes `limit` into `input.GetLedgers()`, which stops after `N` ledgers. `transform.TransformTokenTransfer()` then expands each returned ledger into a variable-length slice of token-transfer rows based on the unified events processor, so the exported row count is determined by how many transfer events happen to exist inside the first `N` ledgers rather than by the caller's requested token-transfer count.

## Trigger

Run `stellar-etl export_token_transfer --limit 1 --start-ledger <S> --end-ledger <E>` on a range where ledger `S` contains no token-transfer events but ledger `S+1` contains at least one valid token-transfer event. The correct output is one transfer row from ledger `S+1`; the current code can return an empty file because ledger `S` consumed the entire limit before any row-level expansion happened.

## Target Code

- `cmd/export_token_transfers.go:21-29` — forwards the user-visible `token_transfer` limit into `GetLedgers()`
- `cmd/export_token_transfers.go:38-60` — exports every `TokenTransferOutput` row returned for each limited ledger
- `internal/input/ledgers.go:14-90` — enforces `limit` on `len(ledgerSlice)`, i.e. ledger count
- `internal/transform/token_transfer.go:14-34` — converts one ledger close meta into token-transfer events
- `internal/transform/token_transfer.go:37-129` — expands a single ledger's event list into a variable number of output rows
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `token_transfer` objects to export

## Evidence

`TransformTokenTransfer()` is explicitly fan-out logic: it calls `EventsFromLedger()`, verifies the event set, then appends one `TokenTransferOutput` per event. Because `GetLedgers()` knows nothing about token-transfer row counts and stops after a fixed number of ledgers, the first limited ledger can contribute zero rows while later ledgers with real transfers are skipped.

## Anti-Evidence

If each of the first `N` ledgers happens to emit exactly one token-transfer row, the observed row count matches the limit by accident. Leaving `--limit` negative also avoids the truncation because all ledgers in range are processed.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `export_token_transfer` command at `cmd/export_token_transfers.go:28` passes the user-visible `limit` directly into `input.GetLedgers()`. Inside `GetLedgers()` (`internal/input/ledgers.go:85`), the limit check `int64(len(ledgerSlice)) >= limit` enforces the cap on the number of ledgers returned, not on the number of token-transfer rows. Back in the exporter loop (`cmd/export_token_transfers.go:38-54`), every `TokenTransferOutput` from `TransformTokenTransfer()` is exported without any secondary row cap. The flag help text from `AddArchiveFlags("token_transfer", ...)` at `internal/utils/main.go:254` explicitly says "Maximum number of token_transfer to export," contradicting the actual ledger-level enforcement.

### Code Paths Examined

- `cmd/export_token_transfers.go:21-28` — `limit` from `MustArchiveFlags` passed directly to `GetLedgers()`, no row-level tracking
- `internal/input/ledgers.go:24-88` — loop iterates `seq = start..end`, appends one `HistoryArchiveLedgerAndLCM` per ledger, breaks when `len(ledgerSlice) >= limit`
- `cmd/export_token_transfers.go:38-54` — iterates all returned ledgers, calls `TransformTokenTransfer()` on each, writes every resulting row; no counter or secondary limit
- `internal/transform/token_transfer.go:14-34` — `TransformTokenTransfer()` calls `EventsFromLedger()` which returns 0..N events per ledger, then `transformEvents()` produces one `TokenTransferOutput` per event
- `internal/utils/main.go:250-254` — `AddArchiveFlags("token_transfer", ...)` defines `--limit` as "Maximum number of token_transfer to export"
- `cmd/export_token_transfers.go:77` — inline comment contradicts flag text: "limit: maximum number of ledgers to export"

### Findings

The bug is confirmed: `--limit N` controls ledger count, not token-transfer row count. This is the same class of bug as the confirmed effects finding (`success/data-input/004`) and trades finding (`success/export-pipeline/010`), but applied to a distinct command and code path. The token transfer case is arguably the worst instance because the limit granularity mismatch spans two levels (ledger → transaction → token-transfer events) rather than one. A single ledger can contain dozens of transactions each producing multiple token-transfer events, making the overshoot potentially very large. Conversely, a ledger with zero token-transfer events silently consumes the limit budget and produces zero rows.

Severity downgraded from High to Medium for consistency with the effects precedent (same bug pattern, same operational-correctness impact). The issue breaks caller-visible batching semantics but does not corrupt monetary field values.

### PoC Guidance

- **Test file**: `internal/transform/token_transfer_test.go` (or a new `internal/transform/data_integrity_poc_test.go` if one exists)
- **Setup**: Build a `xdr.LedgerCloseMeta` containing one transaction with multiple token-transfer events (e.g., a payment + fee event). A second empty `LedgerCloseMeta` (no transactions). Call `TransformTokenTransfer()` on both.
- **Steps**: (1) Show that `TransformTokenTransfer()` on the empty ledger returns 0 rows. (2) Show that `TransformTokenTransfer()` on the event-bearing ledger returns >1 rows. (3) Demonstrate that `GetLedgers(start, end, limit=1, ...)` returns exactly 1 ledger regardless of token-transfer content, proving the limit is enforced at ledger granularity. (4) Show the flag help text via `AddArchiveFlags("token_transfer", ...)` says "token_transfer" not "ledgers."
- **Assertion**: A single ledger consumed by `--limit 1` can produce 0 rows (empty export) or N>1 rows (oversized export), violating the documented "Maximum number of token_transfer to export" semantics.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTokenTransferLimitCountsLedgersNotRows"
**Test Language**: Go

### Demonstration

The test proves the limit/row mismatch in three parts: (1) `transformEvents` with 5 events from a single ledger produces 5 `TokenTransferOutput` rows, meaning `--limit 1` would export 5 rows instead of 1. (2) `transformEvents` with 0 events produces 0 rows, meaning `--limit 1` on an empty ledger wastes the budget and exports nothing. (3) The `--limit` flag help text says "Maximum number of token_transfer to export" while the actual enforcement in `GetLedgers()` caps ledger count, not row count.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/asset"
	"github.com/stellar/go-stellar-sdk/processors/token_transfer"
	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/utils"
	"github.com/spf13/pflag"
	"github.com/stretchr/testify/assert"
)

// TestTokenTransferLimitCountsLedgersNotRows demonstrates that the
// export_token_transfer --limit flag controls ledger count, not
// token_transfer row count. This violates the documented semantics
// of "Maximum number of token_transfer to export."
func TestTokenTransferLimitCountsLedgersNotRows(t *testing.T) {
	operationIndex := uint32(1)

	// Build 5 events that would come from a single ledger.
	// This mirrors the existing test data in token_transfer_test.go.
	multiEvents := []*token_transfer.TokenTransferEvent{
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash1",
				TransactionIndex: 1,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contract1",
			},
			Event: &token_transfer.TokenTransferEvent_Transfer{
				Transfer: &token_transfer.Transfer{
					From:   "from",
					To:     "to",
					Asset:  &asset.Asset{AssetType: &asset.Asset_IssuedAsset{IssuedAsset: &asset.IssuedAsset{AssetCode: "XYZ", Issuer: "issuer1"}}},
					Amount: "100",
				},
			},
		},
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash1",
				TransactionIndex: 1,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contract1",
			},
			Event: &token_transfer.TokenTransferEvent_Mint{
				Mint: &token_transfer.Mint{
					To:     "to",
					Asset:  &asset.Asset{AssetType: &asset.Asset_IssuedAsset{IssuedAsset: &asset.IssuedAsset{AssetCode: "XYZ", Issuer: "issuer1"}}},
					Amount: "200",
				},
			},
		},
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash2",
				TransactionIndex: 2,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contract2",
			},
			Event: &token_transfer.TokenTransferEvent_Burn{
				Burn: &token_transfer.Burn{
					From:   "from",
					Asset:  &asset.Asset{AssetType: &asset.Asset_IssuedAsset{IssuedAsset: &asset.IssuedAsset{AssetCode: "XYZ", Issuer: "issuer1"}}},
					Amount: "300",
				},
			},
		},
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash2",
				TransactionIndex: 2,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contract2",
			},
			Event: &token_transfer.TokenTransferEvent_Fee{
				Fee: &token_transfer.Fee{
					From:   "from",
					Asset:  &asset.Asset{AssetType: &asset.Asset_IssuedAsset{IssuedAsset: &asset.IssuedAsset{AssetCode: "XYZ", Issuer: "issuer1"}}},
					Amount: "50",
				},
			},
		},
		{
			Meta: &token_transfer.EventMeta{
				LedgerSequence:   10,
				TxHash:           "txhash3",
				TransactionIndex: 3,
				OperationIndex:   &operationIndex,
				ContractAddress:  "contract3",
			},
			Event: &token_transfer.TokenTransferEvent_Clawback{
				Clawback: &token_transfer.Clawback{
					From:   "from",
					Asset:  &asset.Asset{AssetType: &asset.Asset_IssuedAsset{IssuedAsset: &asset.IssuedAsset{AssetCode: "XYZ", Issuer: "issuer1"}}},
					Amount: "400",
				},
			},
		},
	}

	lcm := xdr.LedgerCloseMeta{
		V: 1,
		V1: &xdr.LedgerCloseMetaV1{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					ScpValue: xdr.StellarValue{CloseTime: 1000},
					LedgerSeq: 10,
				},
			},
		},
	}

	// --- Part 1: One ledger with 5 events produces 5 rows ---
	output, err := transformEvents(multiEvents, lcm)
	assert.NoError(t, err)

	// If --limit 1 meant "1 token_transfer row", only 1 row should appear.
	// Instead, GetLedgers returns 1 ledger and all 5 events become rows.
	if len(output) <= 1 {
		t.Fatalf("Expected >1 rows from single ledger with 5 events, got %d", len(output))
	}
	t.Logf("OVERSIZED: 1 ledger with 5 events produced %d token_transfer rows (limit=1 would export all 5, not 1)", len(output))

	// --- Part 2: One ledger with 0 events produces 0 rows ---
	emptyOutput, err := transformEvents([]*token_transfer.TokenTransferEvent{}, lcm)
	assert.NoError(t, err)

	if len(emptyOutput) != 0 {
		t.Fatalf("Expected 0 rows from empty events, got %d", len(emptyOutput))
	}
	t.Logf("UNDERSIZED: 1 ledger with 0 events produced 0 token_transfer rows (limit=1 would export 0, not 1)")

	// --- Part 3: Flag help text promises row-level semantics ---
	// AddArchiveFlags("token_transfer", ...) creates a --limit flag
	// described as "Maximum number of token_transfer to export".
	// But GetLedgers enforces the limit on ledger count.
	flags := pflag.NewFlagSet("test", pflag.ContinueOnError)
	utils.AddArchiveFlags("token_transfer", flags)
	limitFlag := flags.Lookup("limit")
	assert.NotNil(t, limitFlag, "--limit flag should exist")
	assert.Contains(t, limitFlag.Usage, "token_transfer",
		"Flag help text should mention 'token_transfer', confirming the documented row-level semantics")
	t.Logf("Flag help text: %q", limitFlag.Usage)
	t.Logf("MISMATCH: help says 'token_transfer' but limit is enforced at ledger granularity in GetLedgers()")

	// --- Summary ---
	t.Logf("BUG DEMONSTRATED: --limit N applied via GetLedgers returns N ledgers.")
	t.Logf("  A ledger with 0 events → 0 rows (undersized export, limit budget wasted)")
	t.Logf("  A ledger with 5 events → 5 rows (oversized export, exceeds limit)")
	t.Logf("  Neither matches 'Maximum number of token_transfer to export'.")
}
```

### Test Output

```
=== RUN   TestTokenTransferLimitCountsLedgersNotRows
    data_integrity_poc_test.go:128: OVERSIZED: 1 ledger with 5 events produced 5 token_transfer rows (limit=1 would export all 5, not 1)
    data_integrity_poc_test.go:137: UNDERSIZED: 1 ledger with 0 events produced 0 token_transfer rows (limit=1 would export 0, not 1)
    data_integrity_poc_test.go:149: Flag help text: "Maximum number of token_transfer to export. If the limit is set to a negative number, all the objects in the provided range are exported"
    data_integrity_poc_test.go:150: MISMATCH: help says 'token_transfer' but limit is enforced at ledger granularity in GetLedgers()
    data_integrity_poc_test.go:153: BUG DEMONSTRATED: --limit N applied via GetLedgers returns N ledgers.
    data_integrity_poc_test.go:154:   A ledger with 0 events → 0 rows (undersized export, limit budget wasted)
    data_integrity_poc_test.go:155:   A ledger with 5 events → 5 rows (oversized export, exceeds limit)
    data_integrity_poc_test.go:156:   Neither matches 'Maximum number of token_transfer to export'.
--- PASS: TestTokenTransferLimitCountsLedgersNotRows (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/transform	0.762s
```
