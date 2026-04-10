# H003: Ledger-entry Parquet rewrites null `sponsor` values to empty strings

**Date**: 2026-04-10
**Subsystem**: data-transform
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

Unsponsored account, account-signer, trustline, and offer rows should keep `sponsor` null/absent consistently across JSON and Parquet exports, while sponsored rows should carry a real `G...` sponsor address. The exporter should not invent `sponsor=""` for rows whose source ledger entries have no sponsor at all.

## Mechanism

The JSON schemas intentionally model `sponsor` as `null.String`, and the transforms produce invalid `null.String{}` values for unsponsored rows via `ledgerEntrySponsorToNullString()` or the default `sponsor` local in `TransformSigners()`. The Parquet schemas then narrow `sponsor` to a required `string`, and each `ToParquet()` method serializes `.String`, turning every unsponsored row into the synthetic value `""`. That collapses the exporter's explicit nullability contract into a fake address-like string field that downstream null-aware analytics can misread.

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

Because a real sponsor is always a Stellar account address, downstream consumers can sometimes special-case `""` as "missing". But that requires format-specific heuristics the JSON export does not require, and it still means the Parquet file no longer preserves the exporter's original null semantics.

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-10
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `ledgerEntrySponsorToNullString()` helper (trustline.go:109-117) returns a zero-value `null.String{}` (Valid=false, String="") when no sponsor exists. All four entity transforms — Account, AccountSigner, Trustline, Offer — use this pattern. The Parquet schema declares `Sponsor string` (required, non-nullable) in all four Parquet structs, and each `ToParquet()` method accesses `.Sponsor.String` directly, which yields `""` for unsponsored rows. JSON serialization of `null.String{Valid: false}` produces `null`, confirming the two formats diverge.

### Code Paths Examined

- `internal/transform/trustline.go:109-117` — `ledgerEntrySponsorToNullString()` returns `null.String{}` (Valid=false) when `SponsoringID()` is nil; returns `null.StringFrom(address)` when sponsor exists
- `internal/transform/account.go:103` — `TransformAccount` stores `Sponsor: ledgerEntrySponsorToNullString(ledgerEntry)` — nullable sponsor
- `internal/transform/account_signer.go:36-39` — `TransformSigners` declares `var sponsor null.String` (zero-value), only sets it via `null.StringFrom()` if signer is sponsored
- `internal/transform/account_signer.go:71-74` — Deletion path follows same pattern for pre-state sponsors
- `internal/transform/offer.go:98` — `TransformOffer` stores `Sponsor: ledgerEntrySponsorToNullString(ledgerEntry)` — nullable sponsor
- `internal/transform/trustline.go:83` — `TransformTrustline` stores `Sponsor: ledgerEntrySponsorToNullString(ledgerEntry)` — nullable sponsor
- `internal/transform/schema.go:113,128,253,280` — All four JSON schemas model `Sponsor null.String`
- `internal/transform/schema_parquet.go:93,108,187,213` — All four Parquet schemas declare `Sponsor string` (required, no optional/repetition tag)
- `internal/transform/parquet_converter.go:122` — `AccountOutput.ToParquet()` writes `ao.Sponsor.String` — yields `""` for null
- `internal/transform/parquet_converter.go:138` — `AccountSignerOutput.ToParquet()` writes `aso.Sponsor.String` — yields `""` for null
- `internal/transform/parquet_converter.go:215` — `TrustlineOutput.ToParquet()` writes `to.Sponsor.String` — yields `""` for null
- `internal/transform/parquet_converter.go:242` — `OfferOutput.ToParquet()` writes `oo.Sponsor.String` — yields `""` for null

### Findings

This is the `null.String` analogue of the confirmed success finding 003 (`min_account_sequence` null.Int → 0 in Parquet). The exact same architectural pattern causes the bug:

1. **Transform layer** intentionally distinguishes sponsored (Valid=true, String="G...") from unsponsored (Valid=false, String="") using `null.String`.
2. **JSON serialization** preserves this: `null.String{Valid: false}` marshals to JSON `null`, while `null.StringFrom("G...")` marshals to `"G..."`.
3. **Parquet schema** uses required `string` (not optional), so there is no way to represent absence.
4. **Parquet converter** reads `.String` directly, which is `""` for invalid `null.String`, collapsing null→empty-string.

This affects **four entity types** (Account, AccountSigner, Trustline, Offer), making it broader in scope than the transaction precondition finding. Downstream BigQuery analytics that filter on `sponsor IS NULL` will get zero results from Parquet-sourced data, while JSON-sourced data returns correct results.

### PoC Guidance

- **Test file**: `internal/transform/data_integrity_poc_test.go` (append to existing PoC file if it exists, otherwise create)
- **Setup**: Construct a minimal `xdr.LedgerEntry` for an account (or any of the four types) with no sponsor extension. Also construct one with a valid sponsor.
- **Steps**: Call the appropriate `Transform*()` function for both entries. Then call `.ToParquet()` on each result. Compare the `Sponsor` field.
- **Assertion**: The unsponsored JSON output should have `Sponsor.Valid == false` (JSON null). The Parquet output should show that `Sponsor == ""`, which is indistinguishable from a legitimate empty string, demonstrating the null collapse. A sponsored entry should show the correct `G...` address in both formats. The test proves the Parquet format loses the null/present distinction that the JSON format preserves.
