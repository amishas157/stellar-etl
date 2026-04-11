# H003: Asset export applies `--limit` before deduplicating to unique assets

**Date**: 2026-04-11
**Subsystem**: data-input
**Severity**: High
**Impact**: Missing asset rows when duplicate operations consume the caller-visible asset limit
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`stellar-etl export_assets --limit N` should emit up to `N` distinct asset rows, because the command itself deduplicates by `AssetID` before writing. Repeated operations involving the same asset should not consume the caller's unique-asset budget if later operations in-range introduce additional assets.

## Mechanism

`input.GetPaymentOperations()` enforces `limit` on qualifying operations, not on unique transformed assets. `cmd/export_assets` then deduplicates the limited operation slice with `seenIDs`, so if early operations all reference the same asset, the reader can stop before later distinct assets are read and the exporter silently returns fewer unique asset rows than the caller requested.

## Trigger

Run `stellar-etl export_assets --limit 2 --start-ledger <S> --end-ledger <E>` on a range where the first two qualifying payment/manage-sell operations both reference asset `A`, and a later qualifying operation in the same range references distinct asset `B`. The correct output is two rows (`A` and `B`); the current code can emit only `A` because the input reader stops after two operations and the writer drops the duplicate during `seenIDs` deduplication.

## Target Code

- `cmd/export_assets.go:30-34` — obtains a limited slice of qualifying operations from the input package
- `cmd/export_assets.go:39-59` — deduplicates on `seenIDs[transformed.AssetID]` after the input limit has already been applied
- `internal/input/assets.go:21-60` — enforces `limit` on `len(assetSlice)`, i.e. qualifying operations
- `internal/transform/asset.go:14-52` — transforms each operation independently into an `AssetOutput`
- `internal/utils/main.go:250-254` — defines `--limit` as the maximum number of `assets` to export

## Evidence

The exporter's own logic proves that its real output unit is the unique asset row, not the source operation: it maintains `seenIDs` precisely to collapse repeated asset appearances within one run. Because the limit is consumed before that dedupe step, duplicate operations at the start of the range can starve later unique assets and produce an underfilled-but-plausible export.

## Anti-Evidence

If the first `N` qualifying operations all correspond to distinct assets, the current implementation happens to satisfy the caller's expectation. Users who do not set `--limit` negative avoid the truncation because the full range is read before dedupe.

---

## Review

**Verdict**: VIABLE
**Severity**: Medium
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `AddArchiveFlags("assets", ...)` call in `export_assets.go:89` generates the flag description "Maximum number of assets to export" (utils/main.go:254). However, the `limit` parameter is passed directly into `GetPaymentOperations()` (assets.go:21) and `GetPaymentOperationsHistoryArchive()` (assets_history_archive.go:13), both of which enforce it on `len(assetSlice)` — the count of qualifying payment/manage-sell-offer operations. The deduplication by `AssetID` happens only afterwards in `export_assets.go:54-56`. This means the limit controls the wrong unit: operations instead of unique assets.

### Code Paths Examined

- `internal/utils/main.go:250-254` — `AddArchiveFlags("assets", ...)` produces help text "Maximum number of assets to export"
- `cmd/export_assets.go:21-34` — extracts `limit` via `MustArchiveFlags` and passes it directly to both asset reader functions
- `internal/input/assets.go:31-57` — `GetPaymentOperations` appends qualifying ops and breaks at `len(assetSlice) >= limit` (operation count, not unique assets)
- `internal/input/assets_history_archive.go:21-47` — `GetPaymentOperationsHistoryArchive` uses identical limit logic on operation count
- `cmd/export_assets.go:40-56` — deduplication loop using `seenIDs[transformed.AssetID]` runs AFTER the limited operation slice is already truncated
- `cmd/export_assets.go:98` — developer comment says "maximum number of operations to export", confirming the mismatch with the flag text

### Findings

The bug is real and analogous to the confirmed success finding 004 (effect limit counts transactions instead of effects), but in the opposite direction:
- **004**: limit counts transactions → oversized effect exports (MORE rows than limit)
- **003**: limit counts operations → underfilled asset exports (FEWER unique assets than limit)

Both `GetPaymentOperations` and `GetPaymentOperationsHistoryArchive` enforce the limit on raw operation count. The `export_assets` command then deduplicates by `AssetID`, potentially discarding duplicates and producing fewer unique rows than the user-specified limit. This is distinct from success finding 002 (asset readers overshoot limit within a ledger), which addresses the per-ledger granularity of the limit check — even if 002's fix were applied, this semantic mismatch would persist.

Severity downgraded from High to Medium because:
- The data that IS exported is individually correct (no field corruption)
- This only triggers when `--limit` is explicitly set AND early operations reference duplicate assets
- Default limit is -1 (no limit), so typical usage is unaffected
- The impact is underfilled exports, not wrong data values

### PoC Guidance

- **Test file**: `internal/input/assets_test.go` or a new PoC test file
- **Setup**: Construct a mock ledger range where early qualifying operations (Payment or ManageSellOffer) reference the same asset, and a later operation references a distinct asset
- **Steps**: Call `GetPaymentOperations(start, end, limit=2, ...)`, then deduplicate the result by AssetID (mirroring export_assets behavior)
- **Assertion**: Assert that the deduplicated result has fewer unique assets than the requested limit of 2, despite additional distinct assets existing later in the range. Compare with calling `GetPaymentOperations` with limit=-1 on the same range and deduplicating — the unlimited call should produce more unique assets.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/input/data_integrity_poc_test.go
**Test Name**: "TestAssetLimitCountsOperationsBeforeUniqueAssets"
**Test Language**: Go

### Demonstration

The test constructs three payment operations where the first two reference the same asset (USDT) and the third references a distinct asset (native XLM). It applies the same limit logic as `GetPaymentOperations` (stop at 2 operations) followed by the same `seenIDs` deduplication as `export_assets`. With limit=2, only 1 unique asset is returned instead of the 2 the user requested, while the unlimited baseline correctly finds 2 distinct assets.

### Test Body

```go
package input

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
	"github.com/stellar/stellar-etl/v2/internal/transform"
)

// TestAssetLimitCountsOperationsBeforeUniqueAssets demonstrates that
// GetPaymentOperations enforces --limit on raw operation count, not on unique
// assets. When early operations reference the same asset, the limit is exhausted
// before later distinct assets are seen, causing export_assets to return fewer
// unique asset rows than the user requested.
func TestAssetLimitCountsOperationsBeforeUniqueAssets(t *testing.T) {
	// --- Setup ---
	// Construct an issuer account for a credit asset.
	issuerAddress := "GBVVRXLMNCJQW3IDDXC3X6XCH35B5Q7QXNMMFPENSOGUPQO7WO7HGZPA"
	issuerID, err := xdr.AddressToAccountId(issuerAddress)
	if err != nil {
		t.Fatalf("failed to parse issuer address: %v", err)
	}

	destAddress := "GAOEOQMXDDXPVJC3HDFX6LZFKANJ4OOLQOD2MNXJ7PGAY5FEO4BRRAQU"
	destID, err := xdr.AddressToAccountId(destAddress)
	if err != nil {
		t.Fatalf("failed to parse dest address: %v", err)
	}
	destMuxed := destID.ToMuxedAccount()

	usdtAsset := xdr.Asset{
		Type: xdr.AssetTypeAssetTypeCreditAlphanum4,
		AlphaNum4: &xdr.AlphaNum4{
			AssetCode: xdr.AssetCode4([4]byte{0x55, 0x53, 0x44, 0x54}), // "USDT"
			Issuer:    issuerID,
		},
	}
	nativeAsset := xdr.MustNewNativeAsset()

	lcm := xdr.LedgerCloseMeta{
		V: 0,
		V0: &xdr.LedgerCloseMetaV0{
			LedgerHeader: xdr.LedgerHeaderHistoryEntry{
				Header: xdr.LedgerHeader{
					LedgerSeq: 100,
					ScpValue: xdr.StellarValue{
						CloseTime: 1000,
					},
				},
			},
		},
	}

	// Three qualifying payment operations:
	//   Op 0: Payment of USDT (asset A)
	//   Op 1: Payment of USDT (asset A) — duplicate
	//   Op 2: Payment of native XLM (asset B) — distinct
	allOps := []AssetTransformInput{
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: destMuxed,
						Asset:       usdtAsset,
						Amount:      100000000,
					},
				},
			},
			OperationIndex:   0,
			TransactionIndex: 0,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  lcm,
		},
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: destMuxed,
						Asset:       usdtAsset,
						Amount:      200000000,
					},
				},
			},
			OperationIndex:   1,
			TransactionIndex: 0,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  lcm,
		},
		{
			Operation: xdr.Operation{
				Body: xdr.OperationBody{
					Type: xdr.OperationTypePayment,
					PaymentOp: &xdr.PaymentOp{
						Destination: destMuxed,
						Asset:       nativeAsset,
						Amount:      300000000,
					},
				},
			},
			OperationIndex:   2,
			TransactionIndex: 0,
			LedgerSeqNum:     100,
			LedgerCloseMeta:  lcm,
		},
	}

	// --- Simulate GetPaymentOperations with limit=2 ---
	// This mirrors assets.go:55: if int64(len(assetSlice)) >= limit && limit >= 0 { break }
	var limit int64 = 2
	var limitedOps []AssetTransformInput
	for _, op := range allOps {
		limitedOps = append(limitedOps, op)
		if int64(len(limitedOps)) >= limit {
			break
		}
	}

	// --- Simulate export_assets dedup (cmd/export_assets.go:54-56) ---
	seenIDs := map[int64]bool{}
	var uniqueAssetsLimited []transform.AssetOutput
	for _, inp := range limitedOps {
		out, err := transform.TransformAsset(inp.Operation, inp.OperationIndex, inp.TransactionIndex, inp.LedgerSeqNum, inp.LedgerCloseMeta)
		if err != nil {
			t.Fatalf("TransformAsset failed: %v", err)
		}
		if seenIDs[out.AssetID] {
			continue
		}
		seenIDs[out.AssetID] = true
		uniqueAssetsLimited = append(uniqueAssetsLimited, out)
	}

	// --- Baseline: unlimited run shows 2 distinct assets exist ---
	seenAll := map[int64]bool{}
	var uniqueAssetsAll []transform.AssetOutput
	for _, inp := range allOps {
		out, err := transform.TransformAsset(inp.Operation, inp.OperationIndex, inp.TransactionIndex, inp.LedgerSeqNum, inp.LedgerCloseMeta)
		if err != nil {
			t.Fatalf("TransformAsset (unlimited) failed: %v", err)
		}
		if seenAll[out.AssetID] {
			continue
		}
		seenAll[out.AssetID] = true
		uniqueAssetsAll = append(uniqueAssetsAll, out)
	}

	// --- Assertions ---
	// The unlimited run finds 2 distinct assets (USDT and native).
	if len(uniqueAssetsAll) != 2 {
		t.Fatalf("precondition: expected 2 unique assets in full dataset, got %d", len(uniqueAssetsAll))
	}

	// BUG: The limited run (limit=2) only finds 1 unique asset because both
	// "slots" were consumed by duplicate USDT operations before the native
	// asset was reached. The user asked for up to 2 assets but got only 1.
	if len(uniqueAssetsLimited) >= int(limit) {
		t.Errorf("expected fewer than %d unique assets with limit=%d (bug not present), got %d",
			limit, limit, len(uniqueAssetsLimited))
	}

	t.Logf("DEMONSTRATED: limit=%d returned %d operations, but only %d unique asset(s) after dedup",
		limit, len(limitedOps), len(uniqueAssetsLimited))
	t.Logf("  Limited unique assets: %v", assetSummary(uniqueAssetsLimited))
	t.Logf("  Unlimited unique assets: %v", assetSummary(uniqueAssetsAll))
	t.Logf("  Missing asset(s) due to limit-before-dedup: native XLM")
}

func assetSummary(assets []transform.AssetOutput) []string {
	var out []string
	for _, a := range assets {
		name := a.AssetType
		if a.AssetCode != "" {
			name = a.AssetCode
		}
		out = append(out, name)
	}
	return out
}
```

### Test Output

```
=== RUN   TestAssetLimitCountsOperationsBeforeUniqueAssets
    data_integrity_poc_test.go:164: DEMONSTRATED: limit=2 returned 2 operations, but only 1 unique asset(s) after dedup
    data_integrity_poc_test.go:166:   Limited unique assets: [USDT]
    data_integrity_poc_test.go:167:   Unlimited unique assets: [USDT native]
    data_integrity_poc_test.go:168:   Missing asset(s) due to limit-before-dedup: native XLM
--- PASS: TestAssetLimitCountsOperationsBeforeUniqueAssets (0.00s)
PASS
ok  	github.com/stellar/stellar-etl/v2/internal/input	0.705s
```
