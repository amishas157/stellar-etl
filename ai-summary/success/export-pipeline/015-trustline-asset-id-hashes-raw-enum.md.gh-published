# 015: Trustline asset_id hashes raw enum name

**Date**: 2026-04-11
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: export-pipeline
**Final review by**: gpt-5.4, high

## Summary

`TransformTrustline()` exports the canonical asset triple (`asset_code`, `asset_issuer`, `asset_type`) but computes `asset_id` from the raw XDR enum label instead of the exported `asset_type` string. As a result, the same Stellar asset gets different `asset_id` values in `trust_lines` versus sibling exports such as assets, offers, trades, and claimable balances, silently breaking cross-table joins on that key.

## Root Cause

`internal/transform/trustline.go` extracts the normalized `assetType` string for the row but ignores it when hashing, passing `asset.Type.String()` into `FarmHashAsset()` instead. `asset.Type.String()` comes from the generated XDR enum map and returns labels like `AssetTypeAssetTypeCreditAlphanum4`, while the rest of the pipeline hashes canonical strings like `credit_alphanum4`.

## Reproduction

During normal `export_ledger_entry_changes` runs, every non-native trustline reaches `TransformTrustline()`. For a credit trustline such as `ETH/<issuer>`, the row exports `asset_type="credit_alphanum4"` but hashes `AssetTypeAssetTypeCreditAlphanum4`, so the emitted `asset_id` disagrees with the `asset_id` generated for the same asset by `transformSingleAsset()` and other production code paths.

## Affected Code

- `internal/transform/trustline.go:TransformTrustline:34-58` — extracts the canonical asset fields but hashes `asset.Type.String()`
- `internal/transform/asset.go:transformSingleAsset:55-63` — sibling asset hashing path uses the canonical extracted `asset_type`
- `internal/transform/claimable_balance.go:TransformClaimableBalance:42-46` — claimable-balance export reuses the canonical asset hashing path
- `internal/transform/offer.go:TransformOffer:34-42` — offer export reuses the canonical asset hashing path

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType`
- **Test language**: `go`
- **How to run**:
  1. `cd <repo-root> && go build ./...`
  2. Create `internal/transform/data_integrity_poc_test.go` with the test body below
  3. Run `go test ./internal/transform/... -run TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType -v`
  4. Observe `trustline=-2311386320395871674` and `asset=4476940172956910889` for the same exported asset triple

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType(t *testing.T) {
	trustlineChange := ingest.Change{
		ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
		Type:       xdr.LedgerEntryTypeTrustline,
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
							Liabilities: xdr.Liabilities{},
						},
					},
				},
			},
		},
	}

	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue:  xdr.StellarValue{CloseTime: 1000},
			LedgerSeq: 10,
		},
	}

	trustlineOutput, err := TransformTrustline(trustlineChange, header)
	if err != nil {
		t.Fatalf("TransformTrustline failed: %v", err)
	}

	assetOutput, err := transformSingleAsset(ethAsset)
	if err != nil {
		t.Fatalf("transformSingleAsset failed: %v", err)
	}

	if trustlineOutput.AssetType != assetOutput.AssetType {
		t.Fatalf("asset_type mismatch: trustline=%q asset=%q", trustlineOutput.AssetType, assetOutput.AssetType)
	}

	if trustlineOutput.AssetCode != assetOutput.AssetCode {
		t.Fatalf("asset_code mismatch: trustline=%q asset=%q", trustlineOutput.AssetCode, assetOutput.AssetCode)
	}

	if trustlineOutput.AssetIssuer != assetOutput.AssetIssuer {
		t.Fatalf("asset_issuer mismatch: trustline=%q asset=%q", trustlineOutput.AssetIssuer, assetOutput.AssetIssuer)
	}

	if trustlineOutput.AssetID != assetOutput.AssetID {
		t.Fatalf(
			"asset_id mismatch for identical exported asset triple: trustline=%d asset=%d trustline_type=%q enum_string=%q",
			trustlineOutput.AssetID,
			assetOutput.AssetID,
			trustlineOutput.AssetType,
			ethTrustLineAsset.Type.String(),
		)
	}
}
```

## Expected vs Actual Behavior

- **Expected**: Trustline rows hash the same canonical `(asset_code, asset_issuer, asset_type)` triple that the row exports, so `trust_lines.asset_id` matches sibling exports for the same asset.
- **Actual**: Trustline rows hash the raw enum string from `asset.Type.String()`, producing a different `asset_id` than other exports for the same asset while still emitting the canonical `asset_type` column.

## Adversarial Review

1. Exercises claimed bug: YES — the test invokes the real `TransformTrustline()` path and compares its output to the production `transformSingleAsset()` hash for the same asset.
2. Realistic preconditions: YES — ordinary credit trustlines in normal exports hit this path without mocks or unreachable setup.
3. Bug vs by-design: BUG — the row already exports canonical asset fields, sibling transforms hash those canonical fields, and no code or documentation defines a trustline-only asset-ID namespace.
4. Final severity: High — this is silent structural corruption of a join key rather than a direct monetary miscalculation.
5. In scope: YES — production exports emit plausible but wrong `asset_id` values for live trustline rows.
6. Test correctness: CORRECT — it proves the exported asset triple matches across both code paths and that only `asset_id` diverges.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Change `internal/transform/trustline.go` to hash `assetType` instead of `asset.Type.String()`, and update the trustline golden expectations so `trust_lines.asset_id` matches the canonical asset IDs used elsewhere in the pipeline.
