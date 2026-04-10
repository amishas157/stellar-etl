# H003: Ledger-entry Parquet rewrites null `sponsor` values to empty strings

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Unsponsored account, account-signer, trustline, and offer rows should keep `sponsor` null/absent consistently across JSON and Parquet exports, while sponsored rows should carry a real `G...` sponsor address. The exporter should not invent `sponsor=""` for rows whose source ledger entries have no sponsor at all.

## Mechanism

The JSON schemas intentionally model `sponsor` as `null.String`, and the transforms produce invalid `null.String{}` values for unsponsored rows via `ledgerEntrySponsorToNullString()` or the default `sponsor` local in `TransformSigners()`. The Parquet schemas then narrow `sponsor` to a required `string`, and each `ToParquet()` method serializes `.String`, turning every unsponsored row into the synthetic value `""`. That collapses the exporter’s explicit nullability contract into a fake address-like string field that downstream null-aware analytics can misread.

## Trigger

Run `export_ledger_entry_changes --write-parquet` over any ledger range containing a mix of sponsored and unsponsored accounts, account signers, trustlines, or offers. The JSON rows preserve `sponsor=null` for unsponsored entries, but the Parquet rows serialize those same entries as `sponsor=""`.

## Target Code

- `internal/transform/trustline.go:ledgerEntrySponsorToNullString:109-117` — shared helper returns invalid `null.String` when no sponsor exists
- `internal/transform/account.go:TransformAccount:86-110` — account rows store `Sponsor: ledgerEntrySponsorToNullString(...)`
- `internal/transform/account_signer.go:TransformSigners:34-46` — signer rows only set `Sponsor` when a signer sponsorship exists
- `internal/transform/account_signer.go:TransformSigners:71-80` — deletion rows preserve the same nullable sponsor behavior
- `internal/transform/offer.go:TransformOffer:79-100` — offer rows store nullable sponsor values
- `internal/transform/schema.go:AccountOutput:96-121` — account JSON schema models `Sponsor null.String`
- `internal/transform/schema.go:AccountSignerOutput:123-134` — account-signer JSON schema models `Sponsor null.String`
- `internal/transform/schema.go:TrustlineOutput:237-258` — trustline JSON schema models `Sponsor null.String`
- `internal/transform/schema.go:OfferOutput:261-283` — offer JSON schema models `Sponsor null.String`
- `internal/transform/schema_parquet.go:AccountOutputParquet:76-101` — account Parquet schema narrows `Sponsor` to `string`
- `internal/transform/schema_parquet.go:AccountSignerOutputParquet:103-114` — account-signer Parquet schema narrows `Sponsor` to `string`
- `internal/transform/schema_parquet.go:TrustlineOutputParquet:171-191` — trustline Parquet schema narrows `Sponsor` to `string`
- `internal/transform/schema_parquet.go:OfferOutputParquet:193-216` — offer Parquet schema narrows `Sponsor` to `string`
- `internal/transform/parquet_converter.go:AccountOutput.ToParquet:105-130` — writes `ao.Sponsor.String`
- `internal/transform/parquet_converter.go:AccountSignerOutput.ToParquet:133-144` — writes `aso.Sponsor.String`
- `internal/transform/parquet_converter.go:TrustlineOutput.ToParquet:199-219` — writes `to.Sponsor.String`
- `internal/transform/parquet_converter.go:OfferOutput.ToParquet:222-245` — writes `oo.Sponsor.String`

## Evidence

`ledgerEntrySponsorToNullString()` in `trustline.go:109-117` returns an invalid `null.String` unless `entry.SponsoringID()` is present, so unsponsored account/trustline/offer rows are explicitly nullable at the transform layer. `TransformSigners()` follows the same pattern with a zero-value `null.String` unless `SponsorPerSigner()` contains the signer. The fixture in `internal/transform/account_signer_test.go:194-215` already contains both `Sponsor: null.String{}` and `Sponsor: null.StringFrom("GBAD...")` rows in the same expected slice, which proves the transform intentionally distinguishes the two states before Parquet conversion flattens them.

## Anti-Evidence

Because a real sponsor is always a Stellar account address, downstream consumers can sometimes special-case `""` as “missing”. But that requires format-specific heuristics the JSON export does not require, and it still means the Parquet file no longer preserves the exporter’s original null semantics.
