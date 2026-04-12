# H003: Unsponsored ledger-entry rows export `sponsor=""` in Parquet instead of null

**Date**: 2026-04-12
**Subsystem**: external-io
**Severity**: High
**Impact**: Non-financial field contains wrong data
**Hypothesis by**: gpt-5.4, high

## Expected Behavior

For accounts, account signers, trustlines, and offers that are not sponsored, the Parquet export should preserve `sponsor` as null. An unsponsored row should not be serialized as an empty string, because downstream analytics use null-ness to distinguish "no sponsor" from a populated sponsor address.

## Mechanism

The transform layer already represents sponsorship with `null.String`, but every affected `ToParquet()` implementation writes `.Sponsor.String` into a plain `string` field. For invalid `null.String` values, that produces `""`, so large classes of unsponsored rows become structurally wrong but still look valid enough to load into BigQuery.

## Trigger

Run `export_ledger_entry_changes --write-parquet` on any batch containing unsponsored accounts, signers, trustlines, or offers. The JSON rows will emit `null` for `sponsor`, while the Parquet rows will emit empty strings.

## Target Code

- `internal/transform/account.go:86-110` — emits `AccountOutput.Sponsor` via `ledgerEntrySponsorToNullString`
- `internal/transform/account_signer.go:34-51` — emits nullable signer sponsors
- `internal/transform/account_signer.go:67-84` — emits nullable sponsors for signer-deletion rows as well
- `internal/transform/trustline.go:68-88` — emits nullable trustline sponsors
- `internal/transform/offer.go:79-100` — emits nullable offer sponsors
- `internal/transform/parquet_converter.go:122,138,215,242` — converts all of those nullable sponsors with `.String`
- `internal/transform/account_signer_test.go:194-204` — current fixtures already distinguish null sponsors from populated sponsor addresses

## Evidence

The source transforms consistently model sponsorship as nullable, which means "no sponsor" is a first-class state in the exported JSON. The Parquet converter drops that state entirely and rewrites it as `""`, even though an empty string is not a valid Stellar account address.

## Anti-Evidence

If every downstream consumer already normalizes `""` back to null, practical impact drops. But that normalization is outside this repository, and the exported Parquet file itself is still carrying the wrong value.
