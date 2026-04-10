# 010: Ledger-entry Parquet rewrites null `sponsor` values to empty strings

**Date**: 2026-04-10
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: data-transform
**Final review by**: gpt-5.4, high

## Summary

The ledger-entry transforms preserve whether `sponsor` is absent by storing it as `null.String` on account, account-signer, trustline, and offer rows. The Parquet path destroys that distinction by converting each nullable value into a required `string`, so every unsponsored row exports `sponsor=""` instead of `null`.

This is a real structural corruption bug in normal export flow. Existing transform fixtures already exercise both sponsored and unsponsored ledger entries, and the revised PoC reproduces the collapse on the real `Transform*()` plus `ToParquet()` path for all four affected row types.

## Root Cause

`ledgerEntrySponsorToNullString()` and `TransformSigners()` intentionally leave unsponsored rows as invalid `null.String` values. The JSON/output schemas model `Sponsor` as nullable, but the Parquet schemas narrow the field to required `string`, and each `ToParquet()` method writes `.Sponsor.String`, which turns `null.String{Valid:false}` into `""`.

## Reproduction

Run `export_ledger_entry_changes --write-parquet` over any ledger range that includes a mix of sponsored and unsponsored accounts, account signers, trustlines, or offers. At the transform/JSON layer, unsponsored rows carry `Sponsor.Valid == false` and serialize as JSON `null`; after Parquet conversion, those same rows contain `sponsor=""`, while sponsored rows still contain real `G...` addresses.

## Affected Code

- `internal/transform/trustline.go:109-117` — `ledgerEntrySponsorToNullString()` returns an invalid `null.String` when no sponsor exists.
- `internal/transform/account.go:86-110` — `TransformAccount()` stores the nullable sponsor returned from the ledger entry helper.
- `internal/transform/account_signer.go:34-46` — `TransformSigners()` leaves unsponsored signer rows as zero-value `null.String`.
- `internal/transform/account_signer.go:71-80` — deleted signer rows preserve the same nullable sponsor behavior.
- `internal/transform/offer.go:79-100` — `TransformOffer()` stores the nullable sponsor returned from the ledger entry helper.
- `internal/transform/schema.go:96-121` — account JSON schema models `Sponsor` as `null.String`.
- `internal/transform/schema.go:123-134` — account-signer JSON schema models `Sponsor` as `null.String`.
- `internal/transform/schema.go:237-258` — trustline JSON schema models `Sponsor` as `null.String`.
- `internal/transform/schema.go:261-283` — offer JSON schema models `Sponsor` as `null.String`.
- `internal/transform/schema_parquet.go:76-101` — account Parquet schema narrows `Sponsor` to required `string`.
- `internal/transform/schema_parquet.go:103-114` — account-signer Parquet schema narrows `Sponsor` to required `string`.
- `internal/transform/schema_parquet.go:171-191` — trustline Parquet schema narrows `Sponsor` to required `string`.
- `internal/transform/schema_parquet.go:193-216` — offer Parquet schema narrows `Sponsor` to required `string`.
- `internal/transform/parquet_converter.go:105-145` — account/account-signer Parquet conversion writes `.Sponsor.String`.
- `internal/transform/parquet_converter.go:199-245` — trustline/offer Parquet conversion writes `.Sponsor.String`.

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestParquetCollapsesNullSponsorToEmptyString`
- **Test language**: `go`
- **How to run**:
  1. `cd /Users/amisha.singla/Documents/amishas157/stellar-etl && go build ./...`
  2. Append the test body below to `internal/transform/data_integrity_poc_test.go`
  3. Run `go test ./internal/transform/... -run TestParquetCollapsesNullSponsorToEmptyString -v`
  4. Observe each subtest report JSON `null` but Parquet `""` for unsponsored rows

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

// TestParquetCollapsesNullSponsorToEmptyString demonstrates that unsponsored
// ledger entries lose their null sponsor semantics on the real transform ->
// parquet path: JSON carries null, while parquet conversion rewrites it to "".
func TestParquetCollapsesNullSponsorToEmptyString(t *testing.T) {
	header := xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}

	t.Run("Account", func(t *testing.T) {
		unsponsoredChange := makeAccountTestInput()
		unsponsoredChange.Pre.Ext = xdr.LedgerEntryExt{}

		unsponsored, err := TransformAccount(unsponsoredChange, header)
		if err != nil {
			t.Fatalf("TransformAccount unsponsored returned error: %v", err)
		}
		if unsponsored.Sponsor.Valid {
			t.Fatalf("expected unsponsored account sponsor to be invalid, got %#v", unsponsored.Sponsor)
		}

		sponsored, err := TransformAccount(makeAccountTestInput(), header)
		if err != nil {
			t.Fatalf("TransformAccount sponsored returned error: %v", err)
		}
		if !sponsored.Sponsor.Valid || sponsored.Sponsor.String == "" {
			t.Fatalf("expected sponsored account fixture to preserve sponsor, got %#v", sponsored.Sponsor)
		}

		parquet := unsponsored.ToParquet().(AccountOutputParquet)
		assertNullCollapse(t, "Account", unsponsored.Sponsor, parquet.Sponsor)

		sponsoredParquet := sponsored.ToParquet().(AccountOutputParquet)
		if sponsoredParquet.Sponsor != sponsored.Sponsor.String {
			t.Fatalf("expected sponsored account parquet sponsor %q, got %q", sponsored.Sponsor.String, sponsoredParquet.Sponsor)
		}
	})

	t.Run("AccountSigner", func(t *testing.T) {
		change, err := makeSignersTestInput(xdr.LedgerEntryChangeTypeLedgerEntryRemoved)
		if err != nil {
			t.Fatalf("makeSignersTestInput returned error: %v", err)
		}

		signers, err := TransformSigners(change, header)
		if err != nil {
			t.Fatalf("TransformSigners returned error: %v", err)
		}

		var unsponsored *AccountSignerOutput
		var sponsored *AccountSignerOutput
		for i := range signers {
			if signers[i].Sponsor.Valid && sponsored == nil {
				sponsored = &signers[i]
			}
			if !signers[i].Sponsor.Valid && unsponsored == nil {
				unsponsored = &signers[i]
			}
		}
		if unsponsored == nil || sponsored == nil {
			t.Fatalf("expected signer fixture to contain both sponsored and unsponsored rows, got %#v", signers)
		}

		parquet := unsponsored.ToParquet().(AccountSignerOutputParquet)
		assertNullCollapse(t, "AccountSigner", unsponsored.Sponsor, parquet.Sponsor)

		sponsoredParquet := sponsored.ToParquet().(AccountSignerOutputParquet)
		if sponsoredParquet.Sponsor != sponsored.Sponsor.String {
			t.Fatalf("expected sponsored signer parquet sponsor %q, got %q", sponsored.Sponsor.String, sponsoredParquet.Sponsor)
		}
	})

	t.Run("Trustline", func(t *testing.T) {
		unsponsoredChange := makeTrustlineTestInput()[0]
		unsponsored, err := TransformTrustline(unsponsoredChange, header)
		if err != nil {
			t.Fatalf("TransformTrustline unsponsored returned error: %v", err)
		}
		if unsponsored.Sponsor.Valid {
			t.Fatalf("expected unsponsored trustline sponsor to be invalid, got %#v", unsponsored.Sponsor)
		}

		sponsoredChange := makeTrustlineTestInput()[0]
		sponsoredChange.Post.Ext = xdr.LedgerEntryExt{
			V: 1,
			V1: &xdr.LedgerEntryExtensionV1{
				SponsoringId: &testAccount3ID,
			},
		}
		sponsored, err := TransformTrustline(sponsoredChange, header)
		if err != nil {
			t.Fatalf("TransformTrustline sponsored returned error: %v", err)
		}
		if !sponsored.Sponsor.Valid || sponsored.Sponsor.String == "" {
			t.Fatalf("expected sponsored trustline fixture to preserve sponsor, got %#v", sponsored.Sponsor)
		}

		parquet := unsponsored.ToParquet().(TrustlineOutputParquet)
		assertNullCollapse(t, "Trustline", unsponsored.Sponsor, parquet.Sponsor)

		sponsoredParquet := sponsored.ToParquet().(TrustlineOutputParquet)
		if sponsoredParquet.Sponsor != sponsored.Sponsor.String {
			t.Fatalf("expected sponsored trustline parquet sponsor %q, got %q", sponsored.Sponsor.String, sponsoredParquet.Sponsor)
		}
	})

	t.Run("Offer", func(t *testing.T) {
		sponsoredChange, err := makeOfferTestInput()
		if err != nil {
			t.Fatalf("makeOfferTestInput returned error: %v", err)
		}

		unsponsoredChange, err := makeOfferTestInput()
		if err != nil {
			t.Fatalf("makeOfferTestInput returned error: %v", err)
		}
		unsponsoredChange.Pre.Ext = xdr.LedgerEntryExt{}

		unsponsored, err := TransformOffer(unsponsoredChange, header)
		if err != nil {
			t.Fatalf("TransformOffer unsponsored returned error: %v", err)
		}
		if unsponsored.Sponsor.Valid {
			t.Fatalf("expected unsponsored offer sponsor to be invalid, got %#v", unsponsored.Sponsor)
		}

		sponsored, err := TransformOffer(sponsoredChange, header)
		if err != nil {
			t.Fatalf("TransformOffer sponsored returned error: %v", err)
		}
		if !sponsored.Sponsor.Valid || sponsored.Sponsor.String == "" {
			t.Fatalf("expected sponsored offer fixture to preserve sponsor, got %#v", sponsored.Sponsor)
		}

		parquet := unsponsored.ToParquet().(OfferOutputParquet)
		assertNullCollapse(t, "Offer", unsponsored.Sponsor, parquet.Sponsor)

		sponsoredParquet := sponsored.ToParquet().(OfferOutputParquet)
		if sponsoredParquet.Sponsor != sponsored.Sponsor.String {
			t.Fatalf("expected sponsored offer parquet sponsor %q, got %q", sponsored.Sponsor.String, sponsoredParquet.Sponsor)
		}
	})
}

func assertNullCollapse(t *testing.T, label string, sponsor interface{}, parquetSponsor string) {
	t.Helper()

	jsonBytes, err := json.Marshal(sponsor)
	if err != nil {
		t.Fatalf("%s sponsor JSON marshal failed: %v", label, err)
	}
	if string(jsonBytes) != "null" {
		t.Fatalf("%s JSON should preserve an unsponsored sponsor as null, got %s", label, string(jsonBytes))
	}
	if parquetSponsor != "" {
		t.Fatalf("%s parquet sponsor should collapse to empty string for unsponsored rows, got %q", label, parquetSponsor)
	}

	t.Errorf("NULL COLLAPSE CONFIRMED: %s JSON sponsor is null, but Parquet sponsor is %q — null semantics lost", label, parquetSponsor)
}
```

## Expected vs Actual Behavior

- **Expected**: Unsponsored ledger-entry rows should remain nullable in Parquet, so they stay distinguishable from sponsored rows without introducing a synthetic string value.
- **Actual**: Unsponsored rows export `sponsor=""` in Parquet even though the transform/JSON layer preserves them as `null`.

## Adversarial Review

1. Exercises claimed bug: YES — the PoC uses real `TransformAccount()`, `TransformSigners()`, `TransformTrustline()`, and `TransformOffer()` fixtures, then runs the production `ToParquet()` converters.
2. Realistic preconditions: YES — sponsored and unsponsored accounts, signers, trustlines, and offers are normal Stellar ledger states.
3. Bug vs by-design: BUG — the exporter already chose nullable semantics in its JSON/output schema, so only the Parquet path is discarding information.
4. Final severity: High — this is silent structural corruption of a non-financial field that changes the meaning of sponsorship analytics.
5. In scope: YES — it is a concrete transform-layer export bug that produces plausible but wrong Parquet data.
6. Test correctness: CORRECT — the PoC proves the transform layer emits distinct nullable states before conversion and that only the Parquet layer collapses them.
7. Alternative explanations: NONE
8. Novelty: NOVEL

## Suggested Fix

Make each Parquet `Sponsor` field nullable, such as `*string` with an optional Parquet tag, and emit `nil` when `Sponsor.Valid` is false. Audit other nullable-to-required Parquet conversions in `parquet_converter.go` so additional `null.*` fields do not silently flatten to zero values.
