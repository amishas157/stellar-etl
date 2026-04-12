# H001: Trustline asset_id hashes the raw XDR enum instead of the exported asset_type

**Date**: 2026-04-12
**Subsystem**: data-integrity
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`TransformTrustline()` should derive `asset_id` from the same canonical asset tuple it exports in the row: `asset_code`, `asset_issuer`, and `asset_type`. A trustline for the same issued asset should therefore get the same `asset_id` as `history_assets`, `history_trades`, and operation detail payloads for that asset.

## Mechanism

`TransformTrustline()` normalizes the human-facing `asset_type` with `asset.Extract(...)`, but then hashes `asset.Type.String()` instead of the normalized string it just exported. For issued assets and pool-share trustlines, the raw enum names (`AssetTypeCreditAlphanum4`, `AssetTypeAssetTypePoolShare`, etc.) differ from the exported canonical values (`credit_alphanum4`, `pool_share`), so `trust_lines.asset_id` silently disagrees with every other table that hashes the canonical tuple.

## Trigger

1. Export any trustline row for a non-native asset, especially a classic credit asset or a pool-share trustline.
2. Export the same asset through `history_assets`, `history_trades`, or operation details.
3. Compare the IDs: the trustline row's `asset_id` will not match the canonical asset hash used everywhere else for the same asset.

## Target Code

- `internal/transform/trustline.go:34-58` — trustline exporter computes `asset_type` from `Extract(...)` but hashes `asset.Type.String()`
- `internal/transform/trustline.go:68-88` — row exports the normalized `asset_type` alongside the mismatched `asset_id`
- `internal/transform/asset.go:55-77` — canonical asset exporters hash the normalized `asset_type`
- `internal/transform/trade.go:45-62` — trade exporter also hashes the normalized `asset_type`

## Evidence

In the trustline path, pool-share rows explicitly export `asset_type = "pool_share"` but hash `"AssetTypeAssetTypePoolShare"` because the third hash input comes from `asset.Type.String()`. Non-pool-share trustlines have the same shape: `Extract(...)` returns strings like `credit_alphanum4`, but the hash input becomes the XDR enum spelling instead. Sibling exporters call `FarmHashAsset(...)` with the normalized `assetType` variable, so the trustline path is a concrete outlier.

## Anti-Evidence

The descriptive trustline columns (`asset_type`, `asset_code`, `asset_issuer`, `liquidity_pool_id`) are still populated, so the row looks plausible and can evade spot checks. This only becomes obvious when downstream joins or deduplication rely on `asset_id` being stable across tables.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced `FarmHashAsset` through all 6 call sites across the codebase. In `asset.go:62`, `trade.go:50,62`, `liquidity_pool.go:44,51`, and `operation.go:384`, the third argument comes from `Extract()` which populates via `AssetTypeToString` (e.g., `"credit_alphanum4"`). Only `trustline.go:57` calls `FarmHashAsset(outputAssetCode, outputAssetIssuer, asset.Type.String())`, where `asset.Type.String()` resolves through the generated `assetTypeMap` which returns `"AssetTypeAssetTypeCreditAlphanum4"` — a completely different string. The `assetType` variable populated by `Extract` on line 52 (or hardcoded on line 50 for pool shares) is never passed to the hash.

### Code Paths Examined

- `internal/transform/trustline.go:52` — `asset.Extract(&assetType, ...)` sets `assetType` to canonical string (e.g., `"credit_alphanum4"`) via `AssetTypeToString` map
- `internal/transform/trustline.go:50` — pool-share branch hardcodes `assetType = "pool_share"`
- `internal/transform/trustline.go:57` — `FarmHashAsset(..., asset.Type.String())` uses the raw XDR enum name instead of `assetType`
- `go-stellar-sdk/xdr/xdr_generated.go:1875-1882` — `assetTypeMap` maps enum ints to `"AssetTypeAssetTypeCreditAlphanum4"` etc.
- `go-stellar-sdk/xdr/asset.go:17-19` — `AssetTypeToString` maps enum to `"credit_alphanum4"` etc.
- `go-stellar-sdk/xdr/asset.go:Extract` — when `typ` is `*string`, sets `*typ = AssetTypeToString[a.Type]`
- `internal/transform/asset.go:57-62` — canonical pattern: `Extract` then `FarmHashAsset(..., outputAssetType)`
- `internal/transform/trade.go:46-50,58-62` — canonical pattern: `Extract` then `FarmHashAsset(..., outputSellingAssetType)`
- `internal/transform/liquidity_pool.go:40-44,47-51` — canonical pattern: `Extract` then `FarmHashAsset(..., assetAType)`
- `internal/transform/operation.go:384` — canonical pattern: `FarmHashAsset(code, issuer, assetType)` where `assetType` from `Extract`

### Findings

Two distinct string maps exist for `xdr.AssetType`:
1. `assetTypeMap` (generated XDR): `{0: "AssetTypeAssetTypeNative", 1: "AssetTypeAssetTypeCreditAlphanum4", 2: "AssetTypeAssetTypeCreditAlphanum12", 3: "AssetTypeAssetTypePoolShare"}` — used by `.String()`
2. `AssetTypeToString` (SDK helper): `{Native: "native", CreditAlphanum4: "credit_alphanum4", CreditAlphanum12: "credit_alphanum12"}` — used by `Extract()`

`FarmHashAsset` concatenates `(code, issuer, type)` and FarmHash64s the result. For a `credit_alphanum4` asset like `USD/GABC...`:
- All other exporters hash: `"USD" + "GABC..." + "credit_alphanum4"`
- Trustline exporter hashes: `"USD" + "GABC..." + "AssetTypeAssetTypeCreditAlphanum4"`

These produce entirely different hash values, making `trust_lines.asset_id` incompatible with `asset_id` in every other exported table. Cross-table JOINs on `asset_id` will silently produce zero matches for trustline rows.

The fail summary entry 018 explicitly acknowledges the trustline path as the outlier: "All other `FarmHashAsset()` callers already normalize through `asset.Extract()` and hash the extracted canonical type string; the trustline path was the outlier."

### PoC Guidance

- **Test file**: `internal/transform/trustline_test.go`
- **Setup**: Construct a `TrustLineEntry` with `AssetTypeAssetTypeCreditAlphanum4` and a known code/issuer. Also construct an `xdr.Asset` with the same code/issuer/type.
- **Steps**: (1) Call `TransformTrustline()` with the trustline entry, extract `AssetID`. (2) Call `transformSingleAsset()` (or directly `FarmHashAsset(code, issuer, "credit_alphanum4")`) with the same asset, extract `AssetID`.
- **Assertion**: Assert that both `AssetID` values are equal. Currently they will NOT be equal, proving the bug. The trustline hash includes `"AssetTypeAssetTypeCreditAlphanum4"` while the asset hash includes `"credit_alphanum4"`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-12
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType"
**Test Language**: Go

### Demonstration

The test constructs a credit_alphanum4 trustline and calls `TransformTrustline()`, then computes the canonical `asset_id` using `FarmHashAsset("ETH", issuer, "credit_alphanum4")`. The trustline produces asset_id `-2311386320395871674` (hashed with raw XDR enum `"AssetTypeAssetTypeCreditAlphanum4"`), while the canonical asset_id is `4476940172956910889` (hashed with `"credit_alphanum4"`). A second test confirms the same mismatch for pool_share trustlines. This proves that `trust_lines.asset_id` is incompatible with every other table's `asset_id` for the same asset.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType demonstrates that
// TransformTrustline() hashes asset.Type.String() (the raw XDR enum name like
// "AssetTypeAssetTypeCreditAlphanum4") instead of the canonical type string
// (like "credit_alphanum4") when computing asset_id. This makes the trustline
// asset_id incompatible with every other table's asset_id for the same asset.
func TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType(t *testing.T) {
	// 1. Construct a trustline for ETH (credit_alphanum4) using existing test fixtures
	trustlineEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeTrustline,
			TrustLine: &xdr.TrustLineEntry{
				AccountId: testAccount1ID,
				Asset:     ethTrustLineAsset,
				Balance:   1000000,
				Limit:     9000000000000000000,
				Flags:     xdr.Uint32(xdr.TrustLineFlagsAuthorizedFlag),
				Ext: xdr.TrustLineEntryExt{
					V: 1,
					V1: &xdr.TrustLineEntryV1{
						Liabilities: xdr.Liabilities{
							Buying:  0,
							Selling: 0,
						},
					},
				},
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
		Pre:        nil,
		Post:       &trustlineEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	// 2. Run the production trustline transform
	trustlineOutput, err := TransformTrustline(change, header)
	if err != nil {
		t.Fatalf("TransformTrustline failed: %v", err)
	}

	// 3. Compute the canonical asset_id the same way asset.go and trade.go do:
	//    FarmHashAsset(code, issuer, canonicalType) where canonicalType = "credit_alphanum4"
	canonicalAssetID := FarmHashAsset("ETH", testAccount3Address, "credit_alphanum4")

	// 4. The trustline's exported asset_type IS "credit_alphanum4" (correct),
	//    but the asset_id was hashed with "AssetTypeAssetTypeCreditAlphanum4" (wrong).
	if trustlineOutput.AssetType != "credit_alphanum4" {
		t.Fatalf("unexpected asset_type: got %q, want %q", trustlineOutput.AssetType, "credit_alphanum4")
	}

	// 5. Show what the trustline actually hashed (the raw XDR enum string)
	rawEnumAssetID := FarmHashAsset("ETH", testAccount3Address, xdr.AssetTypeAssetTypeCreditAlphanum4.String())

	t.Logf("Canonical asset_id (from FarmHashAsset with 'credit_alphanum4'):              %d", canonicalAssetID)
	t.Logf("Trustline asset_id (from TransformTrustline, hashes raw XDR enum):            %d", trustlineOutput.AssetID)
	t.Logf("Raw enum asset_id  (from FarmHashAsset with '%s'): %d", xdr.AssetTypeAssetTypeCreditAlphanum4.String(), rawEnumAssetID)

	// The trustline asset_id should match the canonical asset_id.
	// If it doesn't, the bug is proven: trustline hashes the raw enum instead of the canonical type.
	if trustlineOutput.AssetID != canonicalAssetID {
		t.Errorf("CONFIRMED BUG: trustline asset_id (%d) != canonical asset_id (%d).\n"+
			"Trustline hashes XDR enum '%s' instead of canonical '%s'.",
			trustlineOutput.AssetID, canonicalAssetID,
			xdr.AssetTypeAssetTypeCreditAlphanum4.String(), "credit_alphanum4")
	}

	// Also verify the trustline matches the raw-enum hash (confirming the root cause)
	if trustlineOutput.AssetID != rawEnumAssetID {
		t.Errorf("unexpected: trustline asset_id (%d) doesn't match raw enum hash (%d)", trustlineOutput.AssetID, rawEnumAssetID)
	}
}

// TestTrustlinePoolShareAssetIDHashesRawEnum demonstrates the same bug for
// pool_share trustlines: asset_id is hashed with "AssetTypeAssetTypePoolShare"
// instead of the exported "pool_share" type string.
func TestTrustlinePoolShareAssetIDHashesRawEnum(t *testing.T) {
	// Construct a pool-share trustline using existing test fixtures
	lpEntry := xdr.LedgerEntry{
		LastModifiedLedgerSeq: 100,
		Data: xdr.LedgerEntryData{
			Type: xdr.LedgerEntryTypeTrustline,
			TrustLine: &xdr.TrustLineEntry{
				AccountId: testAccount2ID,
				Asset:     liquidityPoolAsset,
				Balance:   5000000,
				Limit:     1111111111111111111,
				Flags:     xdr.Uint32(xdr.TrustLineFlagsAuthorizedFlag),
				Ext: xdr.TrustLineEntryExt{
					V: 1,
					V1: &xdr.TrustLineEntryV1{
						Liabilities: xdr.Liabilities{
							Buying:  0,
							Selling: 0,
						},
					},
				},
			},
		},
	}

	change := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
		Pre:        nil,
		Post:       &lpEntry,
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	// Run the production trustline transform
	output, err := TransformTrustline(change, header)
	if err != nil {
		t.Fatalf("TransformTrustline failed: %v", err)
	}

	// For pool_share, code and issuer are empty; only the type differs
	canonicalAssetID := FarmHashAsset("", "", "pool_share")
	rawEnumAssetID := FarmHashAsset("", "", xdr.AssetTypeAssetTypePoolShare.String())

	if output.AssetType != "pool_share" {
		t.Fatalf("unexpected asset_type: got %q, want %q", output.AssetType, "pool_share")
	}

	t.Logf("Canonical asset_id (from FarmHashAsset with 'pool_share'):      %d", canonicalAssetID)
	t.Logf("Trustline asset_id (from TransformTrustline):                   %d", output.AssetID)
	t.Logf("Raw enum asset_id  (from FarmHashAsset with '%s'): %d", xdr.AssetTypeAssetTypePoolShare.String(), rawEnumAssetID)

	if output.AssetID != canonicalAssetID {
		t.Errorf("CONFIRMED BUG: pool_share trustline asset_id (%d) != canonical asset_id (%d).\n"+
			"Trustline hashes XDR enum '%s' instead of canonical '%s'.",
			output.AssetID, canonicalAssetID,
			xdr.AssetTypeAssetTypePoolShare.String(), "pool_share")
	}
}
```

### Test Output

```
=== RUN   TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType
    data_integrity_poc_test.go:75: Canonical asset_id (from FarmHashAsset with 'credit_alphanum4'):              4476940172956910889
    data_integrity_poc_test.go:76: Trustline asset_id (from TransformTrustline, hashes raw XDR enum):            -2311386320395871674
    data_integrity_poc_test.go:77: Raw enum asset_id  (from FarmHashAsset with 'AssetTypeAssetTypeCreditAlphanum4'): -2311386320395871674
    data_integrity_poc_test.go:82: CONFIRMED BUG: trustline asset_id (-2311386320395871674) != canonical asset_id (4476940172956910889).
        Trustline hashes XDR enum 'AssetTypeAssetTypeCreditAlphanum4' instead of canonical 'credit_alphanum4'.
--- FAIL: TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType (0.00s)
=== RUN   TestTrustlinePoolShareAssetIDHashesRawEnum
    data_integrity_poc_test.go:152: Canonical asset_id (from FarmHashAsset with 'pool_share'):      721683941183578983
    data_integrity_poc_test.go:153: Trustline asset_id (from TransformTrustline):                   -1967220342708457407
    data_integrity_poc_test.go:154: Raw enum asset_id  (from FarmHashAsset with 'AssetTypeAssetTypePoolShare'): -1967220342708457407
    data_integrity_poc_test.go:157: CONFIRMED BUG: pool_share trustline asset_id (-1967220342708457407) != canonical asset_id (721683941183578983).
        Trustline hashes XDR enum 'AssetTypeAssetTypePoolShare' instead of canonical 'pool_share'.
--- FAIL: TestTrustlinePoolShareAssetIDHashesRawEnum (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.691s
```
