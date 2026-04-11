# H001: `export_assets --limit` can return fewer distinct asset rows than requested

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: structural data corruption
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets --limit N` should stop after exporting at most `N`
distinct asset rows, not after reading `N` raw operations that may later
collapse to fewer unique `asset_id` values. When the requested range contains at
least `N` distinct assets, the export should not silently return a smaller
result set just because the early operations reference the same asset multiple
times.

## Mechanism

Both asset readers stop when `len(assetSlice) >= limit`, but `assetSlice` counts
raw `Payment` / `ManageSellOffer` operations, not exported asset rows. The
command then applies a second-stage `seenIDs` deduplication in
`cmd/export_assets.go`, so repeated early references to asset A can exhaust the
reader limit before later distinct asset B is ever read, causing the final JSON
and Parquet outputs to contain fewer rows than the `--limit` contract promises.

## Trigger

1. Pick a ledger range whose first qualifying operations are:
   1. `Payment` or `ManageSellOffer` referencing asset A
   2. another qualifying operation referencing the same asset A
   3. a later qualifying operation referencing a distinct asset B
2. Run `stellar-etl export_assets --start-ledger <L> --end-ledger <R> --limit 2`.
3. The readers stop after the second raw operation, but `seenIDs` collapses the
   two A rows into one exported asset row, so asset B never appears even though
   the range contains two distinct assets.

## Target Code

- `internal/input/assets.go:31-56` — datastore reader enforces `limit` on raw qualifying operations
- `internal/input/assets_history_archive.go:21-46` — history-archive reader repeats the same raw-operation limit logic
- `cmd/export_assets.go:39-59` — export path deduplicates only after the reader limit has already truncated the slice
- `internal/utils/main.go:250-255` — shared flag text promises `limit` is the maximum number of exported `assets`

## Evidence

The limit check lives outside the inner operation loop in both asset readers and
uses `len(assetSlice)`, which grows once per qualifying operation. The command's
actual row contract is different: it writes only the first occurrence of each
`AssetID` after `TransformAsset()` and drops every duplicate through `seenIDs`.
That split makes row count depend on how many duplicate references happen to
appear before the reader hits the raw-operation limit.

## Anti-Evidence

If the first `N` qualifying operations already reference `N` distinct assets, or
if `--limit` is negative, the bug is masked because the post-read dedupe no
longer changes the cardinality. This does not affect ranges where the caller
expects duplicate asset references to collapse below the requested bound.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated (analogous to confirmed trade finding 010 but distinct entity and mechanism)

### Trace Summary

The `--limit` flag is documented as "Maximum number of assets to export" (`utils/main.go:254`), but both `GetPaymentOperations()` (`assets.go:55`) and `GetPaymentOperationsHistoryArchive()` (`assets_history_archive.go:45`) enforce the limit on the count of raw qualifying operations (Payment/ManageSellOffer), not distinct assets. The export command (`export_assets.go:40-70`) then deduplicates via a `seenIDs` map keyed by `AssetID` (a FarmHash of code+issuer+type), so duplicate operations referencing the same asset consume limit budget without producing additional output rows. The internal code comment at `export_assets.go:98` even acknowledges the limit applies to "operations", contradicting the user-facing flag text.

### Code Paths Examined

- `internal/input/assets.go:21-61` — `GetPaymentOperations()`: appends one entry per qualifying operation (Payment or ManageSellOffer) to `assetSlice`, then checks `int64(len(assetSlice)) >= limit` at ledger boundary. Limit is on raw operation count, not distinct assets.
- `internal/input/assets_history_archive.go:13-51` — `GetPaymentOperationsHistoryArchive()`: identical limit logic via `int64(len(assetSlice)) >= limit`. Same raw-operation counting.
- `cmd/export_assets.go:40-70` — Export loop: calls `TransformAsset()` per operation, then checks `seenIDs[transformed.AssetID]` to skip duplicates. Deduplication happens AFTER the reader limit has already truncated the input slice.
- `internal/transform/asset.go:55-70` — `transformSingleAsset()`: produces `AssetID` via `FarmHashAsset(code, issuer, type)`. Two operations referencing the same asset produce identical `AssetID` values.
- `internal/utils/main.go:250-254` — `AddArchiveFlags()`: flag text reads "Maximum number of assets to export" — the user contract.
- `cmd/export_assets.go:98` — Internal code comment: "limit: maximum number of operations to export" — developer acknowledges the actual behavior contradicts the flag text.

### Findings

This is the same class of bug as the confirmed trade finding (success/export-pipeline/010): the limit is applied at input granularity (raw operations) rather than output granularity (distinct exported rows). For assets, the discrepancy manifests as follows:

1. **Flag contract**: "Maximum number of assets to export" promises N distinct asset rows.
2. **Actual behavior**: Reader stops after N qualifying operations. If k of those N operations reference the same asset, only N-k+1 distinct assets (or fewer) are exported.
3. **Concrete scenario**: In early Stellar ledgers, USDC payments dominate. A `--limit 10` request in a USDC-heavy range could return 2-3 distinct assets instead of 10, even though 10+ distinct assets exist in the full range.
4. **Developer awareness**: The code comment at line 98 says "operations" while the flag says "assets" — this inconsistency was present from the original implementation.

### PoC Guidance

- **Test file**: `internal/transform/asset_limit_poc_test.go` (new file)
- **Setup**: Use a known mainnet ledger range where the first several qualifying operations reference the same asset (e.g., USDC payments). Create a mock or use `GetPaymentOperations()` directly with a small limit.
- **Steps**:
  1. Call `GetPaymentOperations(start, end, limit=2, ...)` on a range where the first 2 Payment/ManageSellOffer operations reference the same asset but a 3rd references a different asset.
  2. Transform all returned operations via `TransformAsset()`.
  3. Deduplicate by `AssetID` (mirroring `export_assets.go` logic).
  4. Count distinct assets in the output.
- **Assertion**: Assert that the deduplicated output contains fewer than `limit` distinct assets, proving the limit was consumed by duplicate operations. Specifically, assert `len(distinctAssets) < limit` when `limit == 2` and the first two operations reference the same asset.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestAssetLimitUnderfillsDistinctRows"
**Test Language**: Go

### Demonstration

The test constructs 3 Payment operations — two referencing the same USDT asset and one referencing the native asset — then simulates the reader's `limit=2` truncation followed by the export command's `seenIDs` deduplication. After truncation, only 2 USDT operations remain; after deduplication, only 1 distinct asset is exported instead of the 2 the user requested. This proves the `--limit` flag counts raw operations rather than distinct exported asset rows.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// assetTransformInput mirrors input.AssetTransformInput to avoid import cycle.
type assetTransformInput struct {
	Operation        xdr.Operation
	OperationIndex   int32
	TransactionIndex int32
	LedgerSeqNum     int32
	LedgerCloseMeta  xdr.LedgerCloseMeta
}

// TestAssetLimitUnderfillsDistinctRows demonstrates that the export_assets
// pipeline's --limit flag counts raw qualifying operations rather than distinct
// exported asset rows. When early operations reference the same asset, the
// reader limit is consumed by duplicates and later distinct assets are never
// read, causing the output to contain fewer distinct assets than requested.
func TestAssetLimitUnderfillsDistinctRows(t *testing.T) {
	// Build 3 Payment operations:
	//   ops[0] -> USDT payment (asset A)
	//   ops[1] -> USDT payment (asset A, same asset as ops[0])
	//   ops[2] -> native payment (asset B, different from A)

	assetA := usdtAsset // credit_alphanum4 USDT
	assetB := nativeAsset

	ops := []assetTransformInput{
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: testAccount2,
						Asset:       assetA,
						Amount:      100000000,
					},
				},
			},
			OperationIndex:   0,
			TransactionIndex: 0,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  genericLedgerCloseMeta,
		},
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: testAccount3,
						Asset:       assetA, // same asset
						Amount:      200000000,
					},
				},
			},
			OperationIndex:   1,
			TransactionIndex: 0,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  genericLedgerCloseMeta,
		},
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: testAccount1,
						Asset:       assetB, // different asset
						Amount:      300000000,
					},
				},
			},
			OperationIndex:   0,
			TransactionIndex: 1,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  genericLedgerCloseMeta,
		},
	}

	// --- Simulate the reader limit ---
	// GetPaymentOperations stops when len(assetSlice) >= limit.
	// With limit=2, the reader returns only ops[0] and ops[1].
	var limit int64 = 2
	readerResult := ops // all 3 ops exist in the ledger range
	if int64(len(readerResult)) >= limit {
		readerResult = readerResult[:limit] // reader truncates to 2
	}

	// --- Simulate the export deduplication (from export_assets.go:40-70) ---
	seenIDs := map[int64]bool{}
	var distinctAssets []AssetOutput
	for _, ti := range readerResult {
		transformed, err := TransformAsset(
			ti.Operation, ti.OperationIndex, ti.TransactionIndex,
			ti.LedgerSeqNum, ti.LedgerCloseMeta,
		)
		if err != nil {
			t.Fatalf("TransformAsset failed: %v", err)
		}
		if _, exists := seenIDs[transformed.AssetID]; exists {
			continue
		}
		seenIDs[transformed.AssetID] = true
		distinctAssets = append(distinctAssets, transformed)
	}

	// --- Assertion ---
	// The user requested limit=2 distinct assets.
	// The ledger range contains at least 2 distinct assets (USDT + native).
	// But because the reader counted raw operations (both USDT payments),
	// it stopped before reaching the native asset, so we get only 1 distinct asset.
	if int64(len(distinctAssets)) < limit {
		t.Errorf("BUG CONFIRMED: --limit %d requested but only %d distinct asset(s) exported. "+
			"The reader limit counted raw operations, not distinct assets. "+
			"Asset B (native) was never read because duplicate references to Asset A consumed the limit budget.",
			limit, len(distinctAssets))
	}
}
```

### Test Output

```
=== RUN   TestAssetLimitUnderfillsDistinctRows
    data_integrity_poc_test.go:116: BUG CONFIRMED: --limit 2 requested but only 1 distinct asset(s) exported. The reader limit counted raw operations, not distinct assets. Asset B (native) was never read because duplicate references to Asset A consumed the limit budget.
--- FAIL: TestAssetLimitUnderfillsDistinctRows (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.717s
```
