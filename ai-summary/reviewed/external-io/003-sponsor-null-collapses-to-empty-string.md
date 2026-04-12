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

---

## Review

**Verdict**: VIABLE
**Severity**: High
**Date**: 2026-04-12
**Reviewed by**: claude-opus-4-6, high
**Novelty**: PASS — not previously investigated

### Trace Summary

The `guregu/null` package's `String` type wraps `sql.NullString` with a `Valid` bool and an inner `String` field. When `Valid` is false (null), the inner `String` is `""`. The transform functions (`ledgerEntrySponsorToNullString` in `trustline.go:109-118`, and inline sponsor construction in `account_signer.go:36-39,71-74`) correctly produce `null.String{Valid: false}` for unsponsored entries. JSON marshaling correctly emits `null`. However, all four `ToParquet()` methods access `.Sponsor.String` which reads the raw inner string, yielding `""` for null sponsors. The Parquet schema uses plain `string` fields without `repetitiontype=OPTIONAL`, so there is no way to represent null in the output.

### Code Paths Examined

- `internal/transform/trustline.go:109-118` — `ledgerEntrySponsorToNullString` returns zero `null.String` (Valid=false, String="") when `SponsoringID()` is nil
- `internal/transform/parquet_converter.go:122` — `AccountOutput.ToParquet()` writes `ao.Sponsor.String` → empty string for null sponsors
- `internal/transform/parquet_converter.go:138` — `AccountSignerOutput.ToParquet()` writes `aso.Sponsor.String` → same issue
- `internal/transform/parquet_converter.go:215` — `TrustlineOutput.ToParquet()` writes `to.Sponsor.String` → same issue
- `internal/transform/parquet_converter.go:242` — `OfferOutput.ToParquet()` writes `oo.Sponsor.String` → same issue
- `internal/transform/schema_parquet.go:93,108,187,213` — All four Parquet structs declare `Sponsor string` without `repetitiontype=OPTIONAL`
- `guregu/null@v4.0.0/string.go:19-21` — `null.String` embeds `sql.NullString`; when `Valid` is false, `.String` is `""`
- `internal/transform/schema_parquet.go:66` — Confirms the parquet library (`xitongsys/parquet-go`) supports `repetitiontype=OPTIONAL`/`REPEATED` in this codebase

### Findings

The bug affects four entity types in Parquet output: accounts, account signers, trustlines, and offers. Every unsponsored row in these tables will have `sponsor=""` instead of null. JSON output is unaffected because `null.String` marshals to JSON `null` when `Valid` is false. The `xitongsys/parquet-go` library supports optional fields via `*string` pointer types with `repetitiontype=OPTIONAL` tags, but the Parquet schema was not written to use this feature. The majority of ledger entries on Stellar are unsponsored, so this affects a large fraction of exported rows. ClaimableBalance is not affected because its Parquet schema is commented out/skipped.

### PoC Guidance

- **Test file**: `internal/transform/parquet_converter_test.go` (create if absent; or append to an existing test file for one of the affected types)
- **Setup**: Construct an `AccountOutput` (or any of the four affected types) with `Sponsor: null.String{}` (zero value, i.e. Valid=false)
- **Steps**: Call `.ToParquet()` on the constructed output, type-assert the result to `AccountOutputParquet`
- **Assertion**: Assert that the returned struct's `Sponsor` field equals `""` — this demonstrates the null collapse. For the fix, assert that using a `*string` field with `repetitiontype=OPTIONAL`, the field is `nil` when the sponsor is null.
