# 014: Unsponsored ledger-entry rows export `sponsor=""` in Parquet instead of null

**Date**: 2026-04-12
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Subsystem**: external-io
**Final review by**: gpt-5.4, high

## Summary

`export_ledger_entry_changes --write-parquet` preserves unsponsored ledger-entry rows as `null.String` in the transform layer, but the Parquet conversion rewrites that missing state to `""`. The result is structurally wrong output for accounts, account signers, trustlines, and offers: JSON exports carry `sponsor: null`, while Parquet exports carry an empty string that was never present in the source data.

This is not just a formatting difference. The Parquet schema uses plain `string` fields for `sponsor`, so the null/empty distinction is unrecoverably lost once the row is converted.

## Root Cause

The transform functions correctly model sponsorship as `null.String`, with `Valid=false` for unsponsored rows. The Parquet conversion layer then reads the embedded `.String` field directly and assigns it into a non-optional `string` column, so every unsponsored value becomes `""`.

## Reproduction

During normal operation, any ledger range containing unsponsored accounts, account signers, trustlines, or offers will hit this path when `export_ledger_entry_changes --write-parquet` is enabled. The JSON path preserves `sponsor` as `null`, but the Parquet path converts the same rows into `sponsor=""`.

## Affected Code

- `internal/transform/account.go:86-110` — `TransformAccount` emits nullable `Sponsor` values via `ledgerEntrySponsorToNullString`
- `internal/transform/account_signer.go:34-51` — `TransformSigners` emits nullable signer sponsors
- `internal/transform/trustline.go:68-88` — `TransformTrustline` emits nullable `Sponsor`
- `internal/transform/trustline.go:109-118` — `ledgerEntrySponsorToNullString` returns an invalid `null.String` when no sponsor exists
- `internal/transform/offer.go:79-100` — `TransformOffer` emits nullable `Sponsor`
- `internal/transform/parquet_converter.go:105-145` — account and signer `ToParquet()` implementations write `.Sponsor.String`
- `internal/transform/parquet_converter.go:199-245` — trustline and offer `ToParquet()` implementations do the same
- `internal/transform/schema_parquet.go:80-114` — account and signer Parquet schemas define `Sponsor` as plain `string`
- `internal/transform/schema_parquet.go:171-216` — trustline and offer Parquet schemas define `Sponsor` as plain `string`

## PoC

- **Target test file**: `internal/transform/data_integrity_poc_test.go`
- **Test name**: `TestSponsorNullCollapsesToEmptyStringInParquet`
- **Test language**: `go`
- **How to run**: Create the target test file with the body below, then run `go test ./internal/transform/... -run TestSponsorNullCollapsesToEmptyStringInParquet -v`.

### Test Body

```go
package transform

import (
	"encoding/json"
	"testing"

	"github.com/stellar/go-stellar-sdk/xdr"
)

func pocHeader() xdr.LedgerHeaderHistoryEntry {
	return xdr.LedgerHeaderHistoryEntry{
		Header: xdr.LedgerHeader{
			ScpValue: xdr.StellarValue{
				CloseTime: 1000,
			},
			LedgerSeq: 10,
		},
	}
}

func requireJSONSponsorNull(t *testing.T, value interface{}) {
	t.Helper()

	raw, err := json.Marshal(value)
	if err != nil {
		t.Fatalf("json.Marshal failed: %v", err)
	}

	var decoded map[string]interface{}
	if err := json.Unmarshal(raw, &decoded); err != nil {
		t.Fatalf("json.Unmarshal failed: %v", err)
	}

	sponsor, ok := decoded["sponsor"]
	if !ok {
		t.Fatalf("expected sponsor field in JSON output: %s", string(raw))
	}
	if sponsor != nil {
		t.Fatalf("expected sponsor to marshal as JSON null, got %v in %s", sponsor, string(raw))
	}
}

func TestSponsorNullCollapsesToEmptyStringInParquet(t *testing.T) {
	header := pocHeader()

	t.Run("account", func(t *testing.T) {
		change := makeAccountTestInput()
		change.Pre.Ext = xdr.LedgerEntryExt{}

		output, err := TransformAccount(change, header)
		if err != nil {
			t.Fatalf("TransformAccount failed: %v", err)
		}
		if output.Sponsor.Valid {
			t.Fatalf("expected unsponsored account output, got %+v", output.Sponsor)
		}
		requireJSONSponsorNull(t, output)

		parquet := output.ToParquet().(AccountOutputParquet)
		if parquet.Sponsor != "" {
			t.Fatalf("expected empty string in parquet sponsor field, got %q", parquet.Sponsor)
		}
	})

	t.Run("account_signer", func(t *testing.T) {
		change, err := makeSignersTestInput(xdr.LedgerEntryChangeTypeLedgerEntryRemoved)
		if err != nil {
			t.Fatalf("makeSignersTestInput failed: %v", err)
		}

		outputs, err := TransformSigners(change, header)
		if err != nil {
			t.Fatalf("TransformSigners failed: %v", err)
		}

		var unsponsored *AccountSignerOutput
		for i := range outputs {
			if !outputs[i].Sponsor.Valid {
				unsponsored = &outputs[i]
				break
			}
		}
		if unsponsored == nil {
			t.Fatal("expected at least one unsponsored signer output")
		}
		requireJSONSponsorNull(t, unsponsored)

		parquet := unsponsored.ToParquet().(AccountSignerOutputParquet)
		if parquet.Sponsor != "" {
			t.Fatalf("expected empty string in parquet sponsor field, got %q", parquet.Sponsor)
		}
	})

	t.Run("trustline", func(t *testing.T) {
		change := makeTrustlineTestInput()[0]

		output, err := TransformTrustline(change, header)
		if err != nil {
			t.Fatalf("TransformTrustline failed: %v", err)
		}
		if output.Sponsor.Valid {
			t.Fatalf("expected unsponsored trustline output, got %+v", output.Sponsor)
		}
		requireJSONSponsorNull(t, output)

		parquet := output.ToParquet().(TrustlineOutputParquet)
		if parquet.Sponsor != "" {
			t.Fatalf("expected empty string in parquet sponsor field, got %q", parquet.Sponsor)
		}
	})

	t.Run("offer", func(t *testing.T) {
		change, err := makeOfferTestInput()
		if err != nil {
			t.Fatalf("makeOfferTestInput failed: %v", err)
		}
		change.Pre.Ext = xdr.LedgerEntryExt{}

		output, err := TransformOffer(change, header)
		if err != nil {
			t.Fatalf("TransformOffer failed: %v", err)
		}
		if output.Sponsor.Valid {
			t.Fatalf("expected unsponsored offer output, got %+v", output.Sponsor)
		}
		requireJSONSponsorNull(t, output)

		parquet := output.ToParquet().(OfferOutputParquet)
		if parquet.Sponsor != "" {
			t.Fatalf("expected empty string in parquet sponsor field, got %q", parquet.Sponsor)
		}
	})
}
```

## Expected vs Actual Behavior

- **Expected**: Unsponsored rows should preserve the missing `sponsor` state in Parquet, just as they do in JSON.
- **Actual**: Unsponsored rows are exported with `sponsor=""` in Parquet, collapsing null into an empty string.

## Adversarial Review

1. Exercises claimed bug: YES — the final PoC uses the real transform helpers for accounts, signers, trustlines, and offers, then compares the resulting JSON and Parquet projections of the same production outputs.
2. Realistic preconditions: YES — unsponsored rows are normal Stellar state, and the PoC uses existing ledger-entry fixtures with sponsorship removed or absent through ordinary XDR fields.
3. Bug vs by-design: BUG — the transform layer already treats missing sponsorship as first-class null data, and there is no repository documentation establishing `""` as an intentional Parquet sentinel.
4. Final severity: High — this is structural data corruption in exported rows, not a direct financial miscalculation.
5. In scope: YES — it is a concrete exporter correctness bug that silently writes plausible but wrong Parquet data.
6. Test correctness: CORRECT — the test proves JSON `null` and Parquet `""` for the same production outputs, so it does not pass because of circular setup or tautological assertions.
7. Alternative explanations: NONE — the observed value change comes directly from `.Sponsor.String` plus non-optional Parquet `string` columns.
8. Novelty: NOT ASSESSED HERE — duplicate handling is delegated to the orchestrator.

## Suggested Fix

Change the affected Parquet schemas to use optional string columns (for example `*string` with `repetitiontype=OPTIONAL`) and update the corresponding `ToParquet()` implementations to emit `nil` when `Sponsor.Valid` is false. That preserves the existing transform semantics and keeps JSON and Parquet aligned.
