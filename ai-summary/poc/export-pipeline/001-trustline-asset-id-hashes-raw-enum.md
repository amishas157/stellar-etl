# H001: Trustline `asset_id` hashes the raw enum name instead of the exported asset type

**Date**: 2026-04-11
**Subsystem**: export-pipeline
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

`trust_lines.asset_id` should be the same deterministic asset key used everywhere else in the export pipeline: hash the canonical `(asset_code, asset_issuer, asset_type)` triple that the row itself exports. A trustline for `credit_alphanum4/ETH/<issuer>` or `pool_share` should therefore receive the same `asset_id` that sibling exports derive for the same asset.

## Mechanism

`TransformTrustline()` is the lone `FarmHashAsset()` call site that hashes `asset.Type.String()` instead of the canonical extracted type string. That means the trustline row exports `asset_type="credit_alphanum4"` or `asset_type="pool_share"` but computes `asset_id` from enum labels like `AssetTypeAssetTypeCreditAlphanum4` / `AssetTypeAssetTypePoolShare`, silently breaking cross-table joins on `asset_id`.

## Trigger

Export any ledger range containing trustlines, then compare a trustline row's `asset_id` against:
1. the hash of that same row's exported `asset_code`, `asset_issuer`, and `asset_type`, or
2. the `asset_id` for the same asset in `claimable_balances`, `history_assets`, `offers`, `trades`, or operation details.

The trustline row will disagree because it hashed the raw enum string instead of the canonical asset type.

## Target Code

- `internal/transform/trustline.go:34-58` — extracts the canonical `assetType`, then hashes `asset.Type.String()` instead
- `internal/transform/asset.go:55-76` — canonical asset hashing path used by sibling exports via `transformSingleAsset()`
- `internal/transform/claimable_balance.go:43-66` — sibling export reuses `transformSingleAsset()` and therefore gets the canonical hash
- `internal/transform/offer.go:34-42,79-90` — offer export uses canonical asset IDs for the same asset namespace

## Evidence

The trustline transform already has the normalized `assetType` string that it writes into the JSON row, but it ignores that value when computing `asset_id`. Every other live `FarmHashAsset()` call site hashes the normalized asset type string instead of the enum's `.String()` form, so `trust_lines` is the outlier in the asset-ID contract.

## Anti-Evidence

If a downstream consumer never joins on `asset_id`, the bug can hide because the row still carries readable `asset_code` / `asset_issuer` / `asset_type` columns. But the schema explicitly exports `asset_id` as the compact join key, and the current trustline value is inconsistent with the rest of the pipeline.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-11
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

Traced the full `FarmHashAsset` call chain from `trustline.go:57` through `asset.go:72-76`. Confirmed that `asset.Type.String()` returns XDR enum names from the generated `assetTypeMap` (e.g., `"AssetTypeAssetTypeCreditAlphanum4"`), while `Asset.Extract()` — used by every other call site — sets the type via `AssetTypeToString` (e.g., `"credit_alphanum4"`). The trustline row exports the canonical form in `AssetType` but hashes the raw enum form in `asset_id`, making the hash inconsistent with all sibling tables.

### Code Paths Examined

- `internal/transform/trustline.go:34-57` — `assetType` is set to canonical string (`"credit_alphanum4"`, `"pool_share"`) via `Extract()` or explicit assignment, but line 57 passes `asset.Type.String()` (raw enum name `"AssetTypeAssetTypeCreditAlphanum4"`) to `FarmHashAsset`
- `internal/transform/asset.go:55-62` — `transformSingleAsset()` calls `asset.Extract(&outputAssetType, ...)` which sets type to `AssetTypeToString[a.Type]` → `"credit_alphanum4"`, then passes that canonical string to `FarmHashAsset`
- `internal/transform/asset.go:72-76` — `FarmHashAsset` concatenates `(code, issuer, type)` and FarmHash-fingerprints it; the type string directly controls the hash value
- `go-stellar-sdk/xdr/xdr_generated.go` — `assetTypeMap` maps `1 → "AssetTypeAssetTypeCreditAlphanum4"` (used by `.String()`)
- `go-stellar-sdk/xdr/asset.go` — `AssetTypeToString` maps `AssetTypeCreditAlphanum4 → "credit_alphanum4"` (used by `Extract()`)
- `internal/transform/trade.go:50,62` — trade export uses `Extract()`-derived type string ✓
- `internal/transform/operation.go:384` — operation export uses extracted `assetType` ✓
- `internal/transform/liquidity_pool.go:44,51` — liquidity pool export uses `Extract()`-derived type ✓
- `internal/transform/offer.go:34-42` — offer export uses `transformSingleAsset()` ✓

### Findings

**The bug is confirmed.** Every `FarmHashAsset` call site except `trustline.go:57` passes the canonical type string (from `Asset.Extract()` → `AssetTypeToString`). The trustline call site passes `asset.Type.String()` which returns the raw XDR enum name. These two strings differ for every asset type:

| Asset Type | `Extract()` canonical | `.Type.String()` raw enum |
|---|---|---|
| Native | `"native"` | `"AssetTypeAssetTypeNative"` |
| Credit Alphanum4 | `"credit_alphanum4"` | `"AssetTypeAssetTypeCreditAlphanum4"` |
| Credit Alphanum12 | `"credit_alphanum12"` | `"AssetTypeAssetTypeCreditAlphanum12"` |
| Pool Share | `"pool_share"` | `"AssetTypeAssetTypePoolShare"` |

This means trustline `asset_id` values will **never** match the `asset_id` for the same asset in any other exported table (`history_assets`, `offers`, `claimable_balances`, `trades`, `operations`, `liquidity_pools`). Cross-table joins on `asset_id` that include trustlines will silently produce zero matches.

The fix is trivial: change `asset.Type.String()` to `assetType` on line 57 of `trustline.go`.

### PoC Guidance

- **Test file**: `internal/transform/trustline_test.go`
- **Setup**: Create a `TrustLineEntry` for a `credit_alphanum4` asset with known code/issuer. Also create the same asset as an `xdr.Asset` and call `transformSingleAsset()` on it.
- **Steps**: Call `TransformTrustline()` on the trustline change, and `transformSingleAsset()` on the equivalent `xdr.Asset`. Compare the `AssetID` fields from both outputs.
- **Assertion**: Assert `trustlineOutput.AssetID == assetOutput.AssetID`. Currently this will FAIL because the trustline hashes `"AssetTypeAssetTypeCreditAlphanum4"` while the asset hashes `"credit_alphanum4"`. Alternatively, compute `FarmHashAsset(code, issuer, "credit_alphanum4")` directly and assert it equals `trustlineOutput.AssetID`.

---

## PoC Attempt

**Result**: POC_PASS
**Date**: 2026-04-11
**PoC by**: claude-opus-4.6, high
**Target Test File**: internal/transform/data_integrity_poc_test.go
**Test Name**: "TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType"
**Test Language**: Go

### Demonstration

The test creates a `credit_alphanum4` trustline (ETH asset), transforms it via `TransformTrustline()`, and compares the resulting `AssetID` against the canonical `FarmHashAsset("ETH", issuer, "credit_alphanum4")` that all other export tables use. The assertion fails because the trustline hashes `"AssetTypeAssetTypeCreditAlphanum4"` (raw enum name via `asset.Type.String()`) instead of `"credit_alphanum4"` (canonical type), producing `asset_id = -2311386320395871674` instead of `4476940172956910889`. This proves cross-table joins on `asset_id` involving trustlines will silently return zero rows.

### Test Body

```go
func TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType(t *testing.T) {
	// 1. Create a credit_alphanum4 trustline and transform it
	trustlineChange := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
		Pre:        nil,
		Post: &xdr.LedgerEntry{
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
		},
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	trustlineOutput, err := TransformTrustline(trustlineChange, header)
	if err != nil {
		t.Fatalf("TransformTrustline failed: %v", err)
	}

	// 2. Compute the canonical asset_id using the same code/issuer/type that
	//    transformSingleAsset (used by offers, claimable_balances, etc.) would use.
	//    The canonical type string for credit_alphanum4 is "credit_alphanum4".
	canonicalAssetID := FarmHashAsset("ETH", testAccount3Address, "credit_alphanum4")

	// 3. Verify the trustline row exports "credit_alphanum4" as its asset_type
	if trustlineOutput.AssetType != "credit_alphanum4" {
		t.Fatalf("Unexpected AssetType: got %q, want %q", trustlineOutput.AssetType, "credit_alphanum4")
	}

	// 4. The key assertion: trustline's asset_id should match the canonical hash.
	//    This FAILS because trustline.go hashes asset.Type.String()
	//    ("AssetTypeAssetTypeCreditAlphanum4") instead of "credit_alphanum4".
	if trustlineOutput.AssetID != canonicalAssetID {
		t.Errorf("Trustline asset_id is inconsistent with canonical hash.\n"+
			"  Trustline AssetID:  %d (hashed from raw enum %q)\n"+
			"  Canonical AssetID:  %d (hashed from canonical type %q)\n"+
			"  Trustline exports AssetType=%q but hashes a different string for asset_id.\n"+
			"  This breaks cross-table joins on asset_id.",
			trustlineOutput.AssetID,
			xdr.AssetTypeAssetTypeCreditAlphanum4.String(),
			canonicalAssetID,
			"credit_alphanum4",
			trustlineOutput.AssetType,
		)
	}
}
```

### Test Output

```
=== RUN   TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType
    data_integrity_poc_test.go:71: Trustline asset_id is inconsistent with canonical hash.
          Trustline AssetID:  -2311386320395871674 (hashed from raw enum "AssetTypeAssetTypeCreditAlphanum4")
          Canonical AssetID:  4476940172956910889 (hashed from canonical type "credit_alphanum4")
          Trustline exports AssetType="credit_alphanum4" but hashes a different string for asset_id.
          This breaks cross-table joins on asset_id.
--- FAIL: TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType (0.00s)
FAIL
FAIL	github.com/stellar/stellar-etl/v2/internal/transform	0.765s
```
