# 018: Trustline asset_id hashes raw enum

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-integrity
**Final review by**: gpt-5.4, high

## Summary

`TransformTrustline()` exports canonical `asset_type` values like `credit_alphanum4` and `pool_share`, but it derives `asset_id` from `asset.Type.String()`, which produces the generated XDR enum names instead. As a result, `trust_lines.asset_id` disagrees with the canonical asset hash used by sibling exporters for the same asset tuple.

I independently reproduced the mismatch with a real trustline transform for both a credit asset and a pool-share trustline. In both cases the exported `asset_id` matched the raw-enum hash and did not match the canonical `(asset_code, asset_issuer, asset_type)` hash.

## Root Cause

`internal/transform/trustline.go` normalizes the human-facing asset tuple first: pool-share trustlines hardcode `assetType = "pool_share"`, and classic trustlines call `asset.Extract(&assetType, &outputAssetCode, &outputAssetIssuer)`, which uses `xdr.AssetTypeToString` and returns canonical values like `credit_alphanum4`.

The function then ignores that normalized `assetType` and computes `outputAssetID := FarmHashAsset(outputAssetCode, outputAssetIssuer, asset.Type.String())`. In the SDK, `asset.Type.String()` comes from the generated `assetTypeMap` and returns names like `AssetTypeAssetTypeCreditAlphanum4` and `AssetTypeAssetTypePoolShare`, so the trustline hash diverges from every caller that hashes the canonical asset-type string.

## Reproduction

During normal operation, exporting any non-native trustline reaches this path. A trustline for `ETH/G...` exports `asset_type = "credit_alphanum4"` but hashes `("ETH", "G...", "AssetTypeAssetTypeCreditAlphanum4")`, while asset, trade, claimable-balance, and operation-detail exporters hash the canonical tuple `("ETH", "G...", "credit_alphanum4")`.

## Affected Code

- `internal/transform/trustline.go:34-58` — derives canonical trustline asset fields, then hashes `asset.Type.String()` instead of the exported `assetType`
- `internal/transform/asset.go:55-63` — canonical asset exporter hashes the normalized `outputAssetType`
- `internal/transform/claimable_balance.go:43-64` — claimable-balance exporter reuses the canonical `transformSingleAsset()` hash
- `internal/transform/operation.go:367-385` — operation details hash the canonical `assetType` extracted from the asset

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType`
- **Test language**: `go`
- **How to run**: Append the test body below to the target test file, then build and run.

### Test Body

```go
package transform

import (
	"testing"

	"github.com/stellar/go-stellar-sdk/ingest"
	"github.com/stellar/go-stellar-sdk/xdr"
)

func TestTrustlineAssetIDHashesRawEnumInsteadOfCanonicalType(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	tests := []struct {
		name            string
		entry           xdr.LedgerEntry
		wantAssetType   string
		wantAssetCode   string
		wantAssetIssuer string
	}{
		{
			name: "credit_alphanum4",
			entry: xdr.LedgerEntry{
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
			wantAssetType:   "credit_alphanum4",
			wantAssetCode:   "ETH",
			wantAssetIssuer: testAccount3Address,
		},
		{
			name: "pool_share",
			entry: xdr.LedgerEntry{
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
								Liabilities: xdr.Liabilities{},
							},
						},
					},
				},
			},
			wantAssetType: "pool_share",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			change := ingest.Change{
				ChangeType: xdr.LedgerEntryChangeTypeLedgerEntryCreated,
				Type:       xdr.LedgerEntryTypeTrustline,
				Post:       &tt.entry,
			}

			output, err := TransformTrustline(change, header)
			if err != nil {
				t.Fatalf("TransformTrustline() error = %v", err)
			}

			if output.AssetType != tt.wantAssetType {
				t.Fatalf("AssetType = %q, want %q", output.AssetType, tt.wantAssetType)
			}

			canonicalAssetID := FarmHashAsset(tt.wantAssetCode, tt.wantAssetIssuer, tt.wantAssetType)
			rawEnumAssetID := FarmHashAsset(tt.wantAssetCode, tt.wantAssetIssuer, tt.entry.Data.TrustLine.Asset.Type.String())

			if output.AssetID != canonicalAssetID {
				t.Fatalf("asset_id mismatch: got %d; want canonical hash %d from (%q,%q,%q); raw-enum hash %d from type %q",
					output.AssetID,
					canonicalAssetID,
					tt.wantAssetCode,
					tt.wantAssetIssuer,
					tt.wantAssetType,
					rawEnumAssetID,
					tt.entry.Data.TrustLine.Asset.Type.String(),
				)
			}
		})
	}
}
```

## Expected vs Actual Behavior

- **Expected**: `trust_lines.asset_id` should hash the same canonical `(asset_code, asset_issuer, asset_type)` tuple that the row exports and that sibling exporters use for the same asset.
- **Actual**: `trust_lines.asset_id` hashes the raw generated XDR enum name, so identical assets receive different IDs across tables.

## Adversarial Review

1. Exercises claimed bug: YES — the test calls the production `TransformTrustline()` path and compares its `asset_id` against the canonical hash for the same exported asset tuple
2. Realistic preconditions: YES — any exported non-native trustline, including standard issued assets and pool-share trustlines, reaches this code path
3. Bug vs by-design: BUG — sibling exporters hash the normalized asset-type string, and no code or docs justify trustlines using a different asset-ID scheme for the same semantic fields
4. Final severity: High — this is structural export corruption that silently breaks cross-table joins and deduplication on `asset_id`, but it does not directly alter monetary fields
5. In scope: YES — it is a concrete silent data-integrity failure in production export code
6. Test correctness: CORRECT — it uses real ledger-entry fixtures, the real transform function, and compares the result against the exact canonical hash contract used elsewhere in the codebase
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Compute `outputAssetID` from the normalized `assetType` variable already being exported:

```go
outputAssetID := FarmHashAsset(outputAssetCode, outputAssetIssuer, assetType)
```

Then update `trustline_test.go` so its expected `AssetID` values match the canonical hashes rather than the raw-enum hashes.
